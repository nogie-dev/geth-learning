
## EVM 구성 요소
```go
var (
		op          OpCode     // current opcode
		jumpTable   *JumpTable = evm.table
		mem                    = NewMemory() // bound memory
		stack                  = newstack()  // local stack
		callContext            = &ScopeContext{
			Memory:   mem,
			Stack:    stack,
			Contract: contract,
		}
		// For optimisation reason we're using uint64 as the program counter.
		// It's theoretically possible to go above 2^64. The YP defines the PC
		// to be uint256. Practically much less so feasible.
		pc   = uint64(0) // program counter
		cost uint64
		// copies used by tracer
		pcCopy    uint64 // needed for the deferred EVMLogger
		gasCopy   uint64 // for EVMLogger to log gas remaining before execution
		logged    bool   // deferred EVMLogger should ignore already logged steps
		res       []byte // result of the opcode execution function
		debug     = evm.Config.Tracer != nil
		isEIP4762 = evm.chainRules.IsEIP4762
	)
```

### 메모리 관리 (Object Pool)

- mem: heap 의 역할
- stack: 연산을 위한 stack 의 역할

반납 방식 차이:

| 구분      | 초기화              | 반납 조건    | 이유                     |
| ------- | ---------------- | -------- | ---------------------- |
| `stack` | `[:0]`           | 무조건      | 크기 고정, 범위 밖 접근 불가      |
| `mem`   | `clear()`+`[:0]` | 16KB 이하만 | 랜덤 접근 가능 → 보안, 대용량은 GC |

---
### Jump Table

### Jump Table

`type JumpTable [256]*operation`

- EVM의 **opcode → 실행 로직(operation)** 을 매핑하는 테이블
- opcode는 1 byte이므로 **최대 256개** 인덱스를 가짐
- 각 인덱스는 해당 opcode의 실행 정보(`operation`)를 가리킴
- Interpreter는 실행 시 `jumpTable[opcode]`로 operation을 찾아 실행함

```go
// JumpTable contains the EVM opcodes supported at a given fork.
type JumpTable [256]*operation
```

### operation

각 opcode 실행에 필요한 메타정보와 실행 함수를 담는 구조체

```go
type operation struct {
	// opcode 실행 함수
	execute     executionFunc

	// 기본적으로 소모되는 gas
	constantGas uint64

	// 실행 상황에 따라 계산되는 추가 gas
	dynamicGas  gasFunc

	// 실행에 필요한 최소 stack 크기
	minStack int

	// stack overflow 방지를 위한 최대 stack 크기
	maxStack int

	// opcode 실행에 필요한 memory 확장 크기 계산
	memorySize memorySizeFunc

	// 해당 opcode가 정의되지 않았는지 여부
	undefined bool
}
```

### Execution Flow

```
bytecode → opcode 읽기
        ↓
jumpTable[opcode]
        ↓
operation.execute()
```

즉 **JumpTable은 EVM interpreter가 opcode를 실제 실행 함수에 빠르게 연결하기 위한 dispatch table 역할을 한다.**

--- 
### Opcode execution loop

- opcode가 STOP, RETURN, SELFDESTRUCT 일 경우 
- opcode 생성 중 오류 발생
- 부모 컨텍스트(parent context)에 의해 `done` 플래그가 설정될 때

루프는 중지됨

- *verkle tree skip*
- contract로 부터 opcode 가져옴
- jumpTable에서 operation 가져옴
	operation 별 가스비 다름
- stack 길이 검증
	- 해당 operation을 실행하는데 필요한 stack의 길이보다 짧으면 Error
- gas 잔고 검증
- 




