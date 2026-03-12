# geth 핵심 Q&A (복습용)

## TX: RPC → TxPool ([[tx-rpc-to-mempool|흐름 분석]])

**Q: checkTxFee의 역할은? 보안 목적인가?**
A: 노드별 가스 상한과 tx fee를 비교하는 것. 보안이 아니라 UX 보호(사용자 실수 방지). 비정상적으로 높은 가스비를 설정한 tx를 걸러냄.

**Q: b.SendTx()에서 b가 인터페이스인 이유는?**
A: 풀 노드와 라이트 노드가 같은 RPC 핸들러를 공유하기 위해. 풀 노드는 txpool.Add() 직접 호출, 라이트 노드는 피어에게 전달. RPC 레이어는 그 차이를 몰라도 되게 설계.

Q: TxPool.Add()에서 서브풀로 분리하는 이유 세 가지?
A:
1. 저장 전략 차이 (legacy는 메모리, blob은 디스크)
2. 정렬/우선순위 기준 차이
3. 용량 제한을 독립적으로 관리

---

## TX: TxPool → Block ([[tx-txpool-to-block|흐름 분석]])

**Q: prio와 normal을 나누는 기준은?**
A: 로컬 tx 여부가 아니라 miner.prio 주소 리스트에 등록된 주소인지. 로컬에서 보내도 리스트에 없으면 normal.

**Q: plain tx와 blob tx가 번갈아 블록에 들어갈 수 있는가?**
A: 가능. 매 루프마다 둘의 tip을 비교해서 높은 쪽을 하나씩 뽑으므로 번갈아 들어갈 수 있음. 분리 처리가 아니라 매 루프마다 경쟁.

**Q: 같은 EOA tx 3개, 첫 번째 가스비가 낮으면?**
A: 첫 번째 tx의 가스비가 이 EOA 묶음 전체의 우선순위를 결정. nonce 순서를 건너뛸 수 없으므로 세 개 다 뒤로 밀림.

**Q: newTxWithMinerFee의 effective tip 공식은?**
A: min(GasFeeCap - baseFee, GasTipCap)
예시: GasFeeCap=100, GasTipCap=5, baseFee=90
→ min(100-90, 5) = min(10, 5) = 5 gwei

**Q: commitTransactions에서 Shift()와 Pop()의 차이는?**
A:
- Shift(): 이 tx 처리 완료, 같은 주소의 다음 tx를 heap에 올림
- Pop(): 이 주소에 문제 있음, 남은 tx 전부 폐기 (nonce 순서 때문)

---

## EVM Execution ([[EVM Execution|흐름 분석]])

**Q: intrinsic gas와 EIP-7623 floorDataGas는 왜 별도로 존재?**
A: intrinsic gas는 기존 가스 계산. floorDataGas는 calldata 전용 하한선. 목적이 다름. floorDataGas는 calldata 대량 사용을 억제해서 blob 사용을 유도하기 위한 별도 규칙.

**Q: Call()에서 대상에 코드가 없으면?**
A: 순수 ETH 전송만 하고 끝 (EVM 실행 없음).

**Q: Call()에서 대상이 존재하지 않고 value도 0이면?**
A: 아무것도 안 하고 리턴 (EIP-158). 존재하지 않는 주소에 0 ETH 보내는 건 무의미.

**Q: Create()에서 CA nonce를 0이 아니라 1로 설정하는 이유?**
A: nonce 0 상태에서 CREATE2로 같은 주소에 재배포하는 공격을 방지하기 위해 (EIP-158).

**Q: sender nonce와 CA nonce의 차이?**
A:
- sender nonce: tx 재실행 방지 (리플레이 방지)
- CA nonce: 같은 주소에 컨트랙트 재배포 방지
- 별개의 값, 별개의 목적

**Q: syscall일 때 Transfer를 건너뛰는 이유?**
A: syscall의 caller는 실제 계정이 아니라 프로토콜이 만든 가상 주소. 잔액이 없으므로 차감하면 안 됨. 시스템 컨트랙트는 데이터 기록 목적이지 ETH 전송이 아님.

---

## Go 설계 패턴

**Q: geth에서 인터페이스 패턴이 반복되는 이유?**
A: "새로운 것이 추가될 때 기존 코드를 수정하지 않기 위해". Backend, SubPool 등 모두 같은 원칙. 구현이 아니라 인터페이스에 의존.

**Q: error를 리턴값으로 처리하는 Go 패턴의 장점?**
A: 에러 처리가 코드 흐름에 명시적으로 드러남. 모든 실패 경로를 if err != nil로 추적 가능.

**Q: 고루틴 + 채널 + select 패턴은 어디서 쓰이나?**
A: worker의 mainLoop(). 새 tx 이벤트, 새 블록 도착 등 여러 이벤트를 동시에 대기하면서 먼저 오는 이벤트를 처리.