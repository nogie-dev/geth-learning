## 1. ApplyTransaction (core/state_processor.go:L233)
- 역할: tx를 EVM 실행에 필요한 형태로 변환하고 실행 진입점 역할

  ### 1.1 ApplyTransactionWithEVM (core/state_processor.go:L162)
  - 역할: 트랜잭션을 Message로 변환 후 EVM에 넘겨 상태 변경을 시도

    #### 1.1.1 ApplyMessage (core/state_transition.go:L213)
    - evm.SetTxContext(): EVM에 tx 정보 세팅
    - → newStateTransition(evm, msg, gp).execute()

      ##### 1.1.1.1 newStateTransition (core/state_transition.go:L255)
      - 역할: 상태 전이를 위한 실행 컨텍스트(evm, msg, gasPool) 조립
      - 구조체 생성 → execute()로 이어짐

---

## 2. execute (core/state_transition.go:L428)
- 역할: 이전 상태(state)와 트랜잭션(msg)을 바탕으로
  EVM을 실행하여 새로운 상태를 도출하는 핵심 함수
- sender nonce(tx 재실행 방지)와 CA nonce(같은 주소 재배포 방지)는 별개

  ### 2.1 preCheck (core/state_transition.go:L316)
  - 역할: 트랜잭션 실행 전 consensus rules 검증

    #### 2.1.1 nonce 검증
    - msg의 nonce == stateDB의 nonce
    - 불일치 시: ErrNonceTooHigh / ErrNonceTooLow

    #### 2.1.2 ParseDelegation (core/tx_setcode.go:L37)
    - EIP-7702 tx일 경우 prefix 제거한 주소와 delegation 여부 반환

    #### 2.1.3 London Patch Validation
    - EIP-1559 가스비 구조 검증 (baseFee, GasFeeCap, GasTipCap)

    #### 2.1.4 Blob Gas Fee Validation
    - blob 가스 시장의 수수료 검증
    - 일반 가스 시장과 blob 가스 시장을 분리 운영
    - 왜 분리: blob이 일반 tx 가스비에 영향 주지 않도록 격리

    #### 2.1.5 EIP-7702 Validation
    - SetCode 트랜잭션 관련 추가 검증

    #### 2.1.6 buyGas (core/state_transition.go:L272)
    - 역할: 트랜잭션 실행에 필요한 가스를 선불로 차감
    - caller의 잔액에서 가스비 차감
    - gasPool에서 가스 확보
    - 실행 후 미사용 가스는 환불됨

  ### 2.2 Intrinsic Gas 계산 및 검증 (core/state_transition.go:L452)
  - 역할: 트랜잭션 실행 전 최소 필요 가스를 계산하고 검증

    #### 2.2.1 기본 가스 설정
    - 컨트랙트 생성 (To == nil) + Homestead 이후: 53,000 gas
    - 일반 전송/호출: 21,000 gas

    #### 2.2.2 Calldata 바이트별 가스 계산
    - zero 바이트 (0x00): 4 gas/byte
    - non-zero 바이트: 16 gas/byte (EIP-2028)
    - 각 연산마다 uint64 overflow 체크

    #### 2.2.3 컨트랙트 생성 추가 비용 (EIP-3860)
    - initcode 크기에 비례한 추가 가스 (word당 InitCodeWordGas)

    #### 2.2.4 Access List 가스 (EIP-2930)
    - 미리 선언한 주소/슬롯에 대해 추가 가스 부과
    - 실행 시 warm access로 처리되어 가스 절약

    #### 2.2.5 EIP-7702 Authorization 가스
    - SetCode 트랜잭션의 authorization 항목당 추가 가스

    #### 2.2.6 최종 intrinsic gas
```
    intrinsicGas = 기본 가스 (21,000 or 53,000)
                 + calldata 가스 (zero/non-zero)
                 + initcode 가스 (컨트랙트 생성 시)
                 + access list 가스
                 + authorization 가스
```

    #### 2.2.7 검증 및 차감
    - gasRemaining < intrinsicGas → 실행 거부
    - EIP-7623 FloorDataGas 체크 (Prague):
      - gasLimit < floorDataGas → 실행 거부
      - 목적: calldata 대량 사용 억제, blob 사용 유도
    - 검증 통과 후 gasRemaining에서 intrinsic gas 차감
    - calldata 비용은 실행 전 선납, 실행 비용은 실행 중 차감

  ### 2.3 Prepare (core/state/statedb.go:L1421)
  - 역할: EVM 실행 전 확실히 접근할 주소/슬롯을 warm 상태로 등록
  - 대상: sender, dst, precompiles, access list, coinbase (EIP-3651)
  - transient storage 초기화 (EIP-1153)

  ### 2.4 Create (core/vm/evm.go:L629)
  - 역할: 컨트랙트 생성 트랜잭션 처리 (To == nil일 때)

    #### 2.4.1 create (core/vm/evm.go:L490)
    1. Snapshot 저장 (실패 시 롤백 지점)
    2. 계정 생성 + 컨트랙트 표시
    3. CA nonce = 1 설정 (EIP-158, 재배포 방지)
    4. caller → 새 주소로 ETH 전송
    5. initcode EVM 실행 → 리턴값이 배포될 바이트코드
    6. 실패 시 snapshot으로 전체 롤백

  ### 2.5 applyAuthorization (EIP-7702 | core/state_transition.go:L626)
  - 역할: EOA에 컨트랙트 코드 위임 설정
  1. 서명 검증 (EOA 소유자 동의 확인)
  2. 기존재 계정이면 가스 차액 환불
  3. nonce 증가 (authorization 재사용 방지)
  4. 제로 주소면 위임 해제, 아니면 위임 설정
  - 효과: EOA가 지정된 컨트랙트 코드로 동작 가능

  ### 2.6 Call (core/vm/evm.go:L239)
  - 역할: 주소에 대한 호출 실행 (To != nil일 때)
  1. 사전 체크 (depth 1024 제한, 잔액 확인) + snapshot 저장
  2. 대상 주소 미존재 시: value 있으면 계정 생성, 없으면 무시 (EIP-158)
  3. caller → addr로 ETH 전송 (syscall이면 건너뜀)
  4. 실행 분기:
     - 프리컴파일 → Go 네이티브 실행 (RunPrecompiledContract)
     - 코드 있음 → evm.Run() (바이트코드 실행) ← interpreter 진입
     - 코드 없음 → 실행 없이 리턴 (순수 ETH 전송)
  5. 실패 시 snapshot으로 롤백
     - REVERT: 상태 롤백, 남은 가스 보존
     - 기타 에러: 상태 롤백 + 가스 전부 소진

  ### 2.7 다음 추적: evm.Run() → Interpreter 루프
  - Call/Create 모두 최종적으로 interpreter에서 바이트코드 실행

## 3.  Run (core/vm/interpreter.go:L428)
- 역할: 

	### 3.1 depth check
	- 함수 진입 시 1 카운팅 (최대 1024), defer로 종료 시 -1 카운팅
	
	3.2 readOnly check
	- staticcall과 같은 경우 하위 호출까지 readOnly 먹임

	3.3 evm return data check
	- 함수 실행 전 이전 실행 결과 초기화
	

