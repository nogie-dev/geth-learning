maxBlockSizeBufferZone = 1MB (miner/worker.go/L96) 
- 왜 block 구성 시 1MB를 남겨두는가 (ToDo)

# TX: TxPool → Block 

## 1. fillTransactions (miner/worker.go:L506)
- 역할: txpool에서 tx를 꺼내 블록에 채우는 전체 과정
- GasPrice, prio 지정 시 데이터 일관성을 위해 RLock

  ### 1.1 Filter Tx (miner/worker.go:L513)
  - 역할: 수집 대상 트랜잭션의 필터링 조건 지정
  - baseFee 이상인 tx만 수집
  - blob과 non-blob을 별도 필터로 분리

  ### 1.2 Pending (core/txpool/txpool.go:L362)
  - 역할: 모든 서브풀에서 pending tx를 주소별로 수집
  - 리턴: map[address] → [tx들] (nonce 순서 보장)
  - 서브풀 내부에서 이미 nonce 정렬된 상태로 나옴

  ### 1.3 Distribution & Prioritizing (miner/worker.go:L537)
  - 역할: pending tx를 prio/normal로 분류 후 각각 우선순위 지정
  - prio 분류 기준: miner.prio 주소 리스트에 등록 여부
    (로컬 tx 여부가 아님)

    #### 1.3.1 newTxWithMinerFee
    - 역할: 마이너의 실제 수익(effective tip) 계산
    - 계산: min(GasFeeCap - baseFee, GasTipCap)
    - GasFeeCap < baseFee → 에러 → 해당 주소 tx 전부 제외
    - 이 값(fees)이 heap 정렬의 기준이 됨

    #### 1.3.2 heap.Init(&heads)
    - 역할: 각 주소의 첫 번째 tx를 effective tip 기준으로 정렬
    - 높은 tip 순, 같으면 도착 시간 빠른 순
    - 이후 Pop()으로 가장 수익 높은 tx부터 블록에 삽입

  ### 1.4 commitTransactions (miner/worker.go:L392)
  - 역할: 정렬된 tx를 하나씩 블록에 삽입하는 메인 루프
  - 루프: 가스 소진 or tx 소진 or interrupt까지 반복

    #### 1.4.1 tx 선택
    - plainTxs.Peek() vs blobTxs.Peek() 비교
    - tip이 높은 쪽을 선택 (타입 무관, 수익 최대화)

    #### 1.4.2 자원 체크
    - 블록 가스 < 21000 → 루프 종료
    - tx 가스 > 남은 가스 → 해당 tx skip (Pop)
    - blob 개수 초과 → blobTxs 전체 제거
    - 블록 사이즈 초과 → 루프 종료

	#### 1.4.3 commitTransaction (miner/worker.go:L333)
	- 역할: 개별 tx를 블록에 넣고 실행
	
	  ##### 1.4.3.1 applyTransaction
	  - blob tx일 경우 사이드카 별도 처리
	  - → core.ApplyTransaction() 호출
	  - 이 안에서 EVM 실행이 일어남 (TODO: EVM 분석 주차)
	  - 실행 결과: receipt (가스 사용량, 로그, 성공/실패)
	
	  ##### 1.4.3.2 실행 후 처리
	  - 성공 시: env.txs에 tx 추가, env.receipts에 receipt 추가
	  - 블록의 가스 사용량 갱신
			
		
	