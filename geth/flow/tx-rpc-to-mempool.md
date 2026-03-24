# TX: RPC → TxPool 
## 1. SendRawTransaction (internal/ethapi/api.go:L1679) 
- input: hexutil.Bytes (서명된 raw tx) 
- RLP 디코딩 → types.Transaction 생성 
- blob tx면 사이드카 분리 
	→ SubmitTransaction() 호출 
	
	### 1.1 SubmitTransaction (internal/ethapi/api.go:L1650) 
	→ checkTxFee() 호출 
	→ b.SendTx() 호출 
	
		#### 1.1.1 checkTxFee (internal/ethapi/api.go:L1620) 
		- 역할: 노드별 gas 상한 지정 가능
		
		#### 1.1.2 SendTx (internal/eth/api_backend.go:L327)
		- 역할: [[구현체 구조|Backend 인터페이스]]를 통해 txpool 진입
		- 왜 api.go 에서 인터페이스 처리로 설계하였는가 → [[구현체 구조]] 참고
		- Track()
			- txPool에 들어가지 못한 tx의 경우 tracking 하다 pool에 자리가 생기면 들어감
			 * blob tx의 경우 용량 문제로 제외됨

			##### 1.1.2.1 Add (core/txpool/txpool.go:L314)
			- 역할: tx를 type에 맞게 분류하고 subpool에 추가하는 함수
			- 왜: 타입을 종류별로 처리하기 위함
				- 저장 전략의 차이
				- 정렬 / 우선순위 기준의 차이
				- blob과 legacy 간 용량의 차이
				
		#### 1.1.3 Recovery(MakeSigner, Sender) (internal/ethapi/api.go:L1596) 
		- 역할: 현 체인 상황에 맞는 서명 형태를 지정한 후 tx의 v, r, s를 바탕으로 sender 추출
		- 어떻게: 체인 설정과 block time, number를 가지고 형태를 단정짓나 (ToDo)

		#### 1.1.4 To (internal/ethapi/api.go:L1602) 
		- 역할: To가 비었을 경우 contract 생성 or To에게 전송
		- EOA 계정을 통해 컨트랙트 배포 시 CREATE (Legacy)로 배포됨