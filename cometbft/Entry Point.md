# receiveRoutine

합의 상태를 변화시키는 메시지들을 **직렬화하여 처리하는 메인 루프**.

## 입력 채널 4가지

| 채널                 | 출처    | WAL 기록 | 역할                           |
| ------------------ | ----- | ------ | ---------------------------- |
| `TxsAvailable`     | 멤풀    | X      | 제안자일 때 블록 생성 트리거             |
| `peerMsgQueue`     | 외부 피어 | O      | Proposal, BlockPart, Vote 수신 |
| `internalMsgQueue` | 본 노드  | O      | 자신이 생성한 제안/투표를 자기 상태에 적용     |
| `timeoutTicker`    | 타이머   | O      | 단계별 타임아웃 만료 처리               |

## 타임아웃 → 상태 전이 매핑

- `timeoutPropose` → enterPrevote (nil 투표, 제안 미도착)
- `timeoutPrevote` → enterPrecommit
- `timeoutPrecommit` → enterPropose (라운드 R+1)
- `timeoutCommit` → enterNewRound (높이 H+1)

---

# handleMsg

## 1. ProposalMessage 처리 (defaultSetProposal)

1. 높이/라운드 일치 검증
2. 이미 해당 라운드에 proposal 있으면 무시
3. POLRound 유효성: -1이거나 `0 ≤ POLRound < Round`
4. 제안자 서명 검증 (`proposal.Verify()`)
5. Proposal 저장 + PartSet 초기화 (블록 파트 수집 준비)

## 2. BlockPartMessage 처리 (addProposalBlockPart)

1. 높이 일치 검증
2. ProposalBlockParts가 nil이 아닌지 확인 (Proposal 수신 후 PartSet이 초기화되어 있어야 함)
3. Part 추가 시 **Merkle proof로 무결성 검증**
4. 총 블록 크기가 `MaxBytes` 이내인지 확인
5. 모든 파트 수집 완료 시 → 블록 디코딩 → `cs.ProposalBlock`에 저장 → handleCompleteProposal로 진입

## 3. handleCompleteProposal

1. Polka 확인, 블록 null 체크, 라운드 최신화 검증
2. validRound, Block, BlockParts 업데이트
3. 라운드 진행 상황 및 과거 제안/블록 형성 검증

### → doPrevote (defaultDoPrevote)

- 잠긴 블록이 있는데 제안 블록과 다르면 → **nil sign**
- 제안 블록이 없으면 → **nil sign**
- `ValidateBlock` 실패 → **nil sign**
- 앱에 `ProcessProposal` 요청 → reject이면 **nil sign**
- 모두 통과하면 → **해당 블록에 prevote sign**

### → Polka 달성 시 최적화 분기

Propose 단계 이하이고 proposal 완성 상태면:

- `enterPrevote` → 이미 2/3 달성이면 바로 `enterPrecommit`

Commit 단계면:

- `tryFinalizeCommit` 시도

---

# enterPrecommit → tryFinalizeCommit 까지의 플로우

이 흐름의 핵심은 **RoundStep이 어떻게 변해가며 최종 커밋에 도달하는가**이다.

## Step 1: enterPrecommit 진입

prevote 결과에 따라 세 갈래로 분기:

**(a) 2/3 polka 없음** → nil precommit 서명 → RoundStep을 `Precommit`으로 업데이트 → 종료

**(b) 2/3가 nil** → 잠금 해제 (LockedRound=-1, LockedBlock=nil) → nil precommit 서명 → RoundStep을 `Precommit`으로 업데이트 → 종료

**(c) 2/3가 특정 블록 v** → 잠금 갱신 (LockedRound=round, LockedBlock=v) → v에 precommit 서명 → RoundStep을 `Precommit`으로 업데이트

## Step 2: Precommit 투표 수집

다른 노드들의 precommit이 도착하면서:

- **2/3 precommit 수신** (any) → RoundStep이 `PrecommitWait`로 전이, `timeoutPrecommit` 시작
- **2/3 precommit이 특정 블록 v에 집중** → `enterCommit` 진입

## Step 3: enterCommit

- RoundStep을 `Commit`으로 업데이트
- 커밋 대상 블록이 이미 있으면 → **tryFinalizeCommit** 즉시 호출
- 블록이 아직 없으면 → 블록 파트 수집 대기 (파트가 다 모이면 handleCompleteProposal에서 tryFinalizeCommit 호출)

## Step 4: tryFinalizeCommit

- 블록 검증 + 2/3 precommit 서명 최종 확인
- 성공 시 → **finalizeCommit** → 블록을 체인에 적용
- RoundStep이 `NewHeight`로 전이 → `timeoutCommit` 후 다음 높이 시작
