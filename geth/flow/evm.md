## EVM Interpreter

### 구성 요소
- **stack**: 연산용 스택 (최대 1024 depth)
- **mem**: 동적 메모리 (heap 역할, 바이트 단위 랜덤 접근)
- **pc**: 프로그램 카운터 (현재 실행 위치)
- **jumpTable**: opcode → operation 매핑 테이블 (256개)

### Jump Table & Operation 매핑
```
jumpTable[256]*operation

opcode 0x01 (ADD)  → operation{ execute: opAdd,  constantGas: 3,  minStack: 2 }
opcode 0x55 (SSTORE) → operation{ execute: opSstore, dynamicGas: gasSStore, minStack: 2 }
opcode 0xF1 (CALL) → operation{ execute: opCall, dynamicGas: gasCall, minStack: 7 }
```

각 operation은 실행 함수, 가스 비용, 스택 요구사항을 담고 있음.
하드포크별로 다른 jumpTable을 사용 (같은 opcode라도 포크마다 가스비/동작이 다를 수 있음).

매핑 위치: `core/vm/jump_table.go`에서 포크별로 테이블 생성.

### 실행 루프
```
for {
    1. contract에서 opcode 읽기 (pc 위치)
    2. jumpTable[opcode]로 operation 조회
    3. stack 길이 검증 (minStack/maxStack)
    4. 가스 계산: constantGas + dynamicGas + memoryExpansion
    5. 가스 잔고 검증 (부족하면 ErrOutOfGas)
    6. operation.execute() 실행
    7. pc++
}
```

### 루프 종료 조건
- STOP, RETURN, REVERT, SELFDESTRUCT opcode
- 실행 중 에러 발생 (가스 부족, 스택 오류 등)
- 부모 컨텍스트의 done 플래그 (외부에서 취소)