
## 1. ApplyTransaction (core/state_processor.go:L233)
- 역할: tx를 EVM 실행에 필요한 형태로 변환하고 실행 진입점 역할

  ### 1.1 ApplyTransactionWithEVM (core/state_processor.go:L162)
  - 역할: 트랜잭션을 Message로 변환 후 EVM에 넘겨 상태 변경을 시도

    #### 1.1.1 ApplyMessage (core/state_transition.go:L213)
    - evm.SetTxContext(): EVM에 tx 정보 세팅 (TODO: 상세)
    - → newStateTransition(evm, msg, gp).execute()

      ##### 1.1.1.1 newStateTransition (core/state_transition.go:L255)
      - 역할: 상태 전이를 위한 실행 컨텍스트(evm, msg, gasPool) 조립
      - 실제 로직 없이 구조체 생성 → execute()로 이어짐

---

## 2. execute (core/state_transition.go:L428)
- 역할: 이전 상태(state)와 트랜잭션(msg)을 바탕으로
  EVM을 실행하여 새로운 상태를 도출하는 핵심 함수

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
    - 왜 분리? (TODO: blob이 일반 tx 가스비에 영향 주지 않도록 격리)

    #### 2.1.5 EIP-7702 Validation
    - SetCode 트랜잭션 관련 추가 검증

    #### 2.1.6 buyGas (core/state_transition.go:L272)
    - 역할: 트랜잭션 실행에 필요한 가스를 선불로 차감
    - caller의 잔액에서 가스비 차감
    - gasPool에서 가스 확보
    - 실행 후 미사용 가스는 환불됨