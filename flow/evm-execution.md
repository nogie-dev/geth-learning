## 1. ApplyTransaction (core/state_processor.go:L233)
- 역할: Tx를 EVM에서 사용할 msg 형태로 변환 및 

	### 1.1 ApplyTransactionWithEVM (core/state_processor.go:L162)
	- 역할: state database에 추가하고, 상태 데이터베이스에 트랜잭션을 적용하려고 시도함

		#### 1.1.1 ApplyMessage (core/state_transition.go:L213)
			- evm.SetTxContext(): EVM에 tx 정보 세팅 (TODO: 상세) 
			- → newStateTransition(evm, msg, gp).execute() ← 핵심
			
			1.1.1.1 newStateTransition
			- 역할: newStateTransition은 새로운 상태(state)를 계산하기 위한 실행 컨텍스트를 세팅하는 함수다