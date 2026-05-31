# RTT-Aware Multipath Design Draft

## 목적

이 문서는 현재 확인된 `processEv` RTT plumbing을 기반으로, UEC multipath의 RTT-aware 확장 방향을 정리한다.  
이번 단계는 설계 문서화만 수행하며 C++ 코드는 아직 변경하지 않는다.

우선순위:
1. 1차 구현 후보: `RTT-Bitmap`
2. 2차 구현 후보: `RTT-Mixed`

---

## 1) 현재 `processEv` 구조

현재 인터페이스 구조는 behavior-preserving overload 형태다.

- 2-arg pure virtual:
  - `processEv(uint16_t path_id, PathFeedback feedback)`
- 3-arg overload:
  - `processEv(uint16_t path_id, PathFeedback feedback, simtime_picosec raw_rtt)`
- base class 기본 동작:
  - 3-arg overload는 `raw_rtt`를 사용하지 않고 2-arg `processEv(path_id, feedback)`로 forward

의미:
- 기존 `UecMpBitmap`, `UecMpReps`, `UecMpRepsLegacy`, `UecMpOblivious`, `UecMpMixed` behavior는 유지된다.
- RTT-aware class만 선택적으로 3-arg를 override해서 `raw_rtt`를 사용하면 된다.

---

## 2) `raw_rtt` source 정리

- ACK path:
  - `raw_rtt`를 normal RTT sample로 사용 가능
- NACK path:
  - `raw_rtt` 계산/전달 가능
  - RTT-aware 알고리즘에서는 normal RTT sample이 아니라 bad feedback elapsed sample로 해석 필요
- TIMEOUT path:
  - `timeInf` sentinel 전달
  - normal RTT sample로 사용 금지

---

## 3) sentinel 주의사항

- 현재 `timeInf` 값은 `0`
- RTT-aware 알고리즘에서 `raw_rtt == 0`을 low RTT로 해석하면 안 된다.
- `raw_rtt == 0`은 no RTT sample(또는 invalid sample)로 처리해야 한다.

권장 처리 규칙:
- `raw_rtt > 0`일 때만 RTT 통계(`srtt`, `min_rtt`, `base_rtt`) 업데이트
- `raw_rtt == 0`이면 RTT 통계 업데이트를 건너뛰고 feedback type 기반 패널티/감점 로직만 적용

---

## 4) `RTT-Bitmap` 설계 초안

핵심 아이디어:
- 기존 `UecMpBitmap`의 path 선호/회피 구조는 유지
- path health를 `feedback` 중심에서 `feedback + RTT inflation`으로 확장

### 상태 변수(안)

- `base_rtt[path]` 또는 `min_rtt[path]`
  - 해당 path의 장기 기준 RTT
- `srtt[path]`
  - EWMA 기반 단기 RTT
- `penalty[path]`
  - path 선택 억제 가중치

### 계산(안)

- ACK + 유효 RTT sample(`raw_rtt > 0`)일 때:
  - `srtt[path] = (1-a) * srtt[path] + a * raw_rtt`
  - `base_rtt[path] = min(base_rtt[path], raw_rtt)` 또는 장기 완만 업데이트
  - `inflation = srtt[path] / max(base_rtt[path], eps)` 또는 `srtt - base_rtt`
- penalty 업데이트:
  - inflation이 임계값보다 높으면 penalty 증가
  - 정상 ACK가 지속되면 penalty 점진 감소
  - NACK/TIMEOUT은 inflation과 무관하게 강한 penalty 증가 가능

### `UecMpBitmap`과의 관계

- `nextEntropy()`의 골격은 최대한 기존 유지
- path 선택 시 bitmap 후보 중 penalty가 낮은 경로를 우선
- 동률 시 기존 entropy/random tie-break 유지

### 모든 path가 bad일 때 fallback

- hard lockout 금지
- 최소 1개 path는 항상 선택 가능하도록 보장
- fallback 우선순위 예:
  1. penalty 최소 path
  2. 최근 성공 ACK 존재 path
  3. 균등 random exploration

---

## 5) `RTT-Mixed` 설계 초안

핵심 아이디어:
- `UecMpMixed`의 혼합 전략을 유지하면서 RTT 기반 whitelist/penalty 계층을 추가

### 구성 요소(안)

- low RTT path whitelist
  - 최근 RTT quality가 좋은 path 집합
- high RTT path penalty
  - inflation이 높은 path 억제
- freshness TTL
  - 오래된 RTT sample 무효화 시간
- reuse budget
  - 같은 path 연속 사용 상한
- exploration 비율
  - whitelist 외 path도 일정 비율로 탐색

### 동작 개요(안)

1. whitelist가 비어있지 않으면 whitelist 우선 선택
2. 단, reuse budget 초과 시 다른 후보 탐색
3. 일정 확률(예: 5~15%)로 exploration 수행
4. RTT sample freshness 만료 시 해당 path score를 중립 방향으로 완화

### `UecMpMixed`와의 관계

- 기존 mixed의 balancing 성격 유지
- RTT로 candidate ordering만 조정
- feedback(`ACK`/`NACK`/`TIMEOUT`) 기반 기존 안정장치는 그대로 유지

---

## 6) 구현 순서 제안

1. `RTT-Bitmap` class 추가
   - 기존 class와 분리된 신규 class로 도입
2. command-line option 추가
   - 예: `-load_balancing_algo rtt_bitmap` (이름은 구현 시점 확정)
3. `one.cm` smoke test
   - 동작/회귀 기본 확인
4. `perm_16n_16c_2MB.cm` baseline 비교
   - `latest finish`, `Rtx`, `NACKs` 우선 확인
5. `perm_random_1024n_1024c_seed42_0u_16777216b.cm` 비교
   - stress workload에서 안정성/재전송 경향 확인
6. `RTT-Mixed` 구현 및 동일 실험 반복

---

## 7) 실험에서 비교할 metric

- `latest finish time`
- `Rtx`
- `NACKs`
- `ACKs`
- `tail gap`
  - 예: 알고리즘별 `latest finish` 차이
- 알고리즘별 stability
  - 동일 seed/동일 matrix 반복 시 지표 변동 폭

권장 비교 표:
- baseline(`bitmap`, `mixed`, `oblivious`) 대비
- RTT-aware(`rtt_bitmap`, 이후 `rtt_mixed`) 상대 개선/악화

