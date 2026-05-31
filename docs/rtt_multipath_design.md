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

## 3-1) NACK `raw_rtt` 반영 정책 초안

- NACK `raw_rtt`는 normal ACK RTT sample과 섞지 않는다.
- NACK는 RTT 값보다 bad feedback event로 우선 처리한다.
- NACK `raw_rtt`는 필요 시 다음 중 하나로 제한적으로 사용한다.
  - `bad feedback elapsed sample` 전용 상태에 별도 저장
  - ACK RTT 계열보다 낮은 가중치로만 참고
- 1차 `RTT-Bitmap` 구현 기본값:
  - NACK `raw_rtt`를 `srtt` 업데이트에 사용하지 않는다.

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
- invalid RTT sample 처리 규칙:
  - `raw_rtt == 0`이면 invalid/no RTT sample로 취급
  - invalid RTT sample은 `min_rtt/srtt/base_rtt` 업데이트에 사용하지 않음
  - TIMEOUT에서 전달되는 `timeInf == 0`은 low RTT signal이 아니며, 낮은 RTT로 해석 금지

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
5. `raw_rtt == 0`이면 whitelist/good path 판정에 사용하지 않음
6. invalid RTT sample은 good path pool에 넣지 않음
7. TIMEOUT sentinel(`timeInf == 0`)은 low RTT signal이 아님

### `UecMpMixed`와의 관계

- 기존 mixed의 balancing 성격 유지
- RTT로 candidate ordering만 조정
- feedback(`ACK`/`NACK`/`TIMEOUT`) 기반 기존 안정장치는 그대로 유지

---

## 6) 구현 순서 제안

1. `RTT-Bitmap` class 추가
   - 기존 class와 분리된 신규 class로 도입
2. command-line option 추가
   - 후보: `-load_balancing_algo rtt_bitmap`
   - 후보: `-load_balancing_algo rtt_mixed`
   - 기존 `bitmap`/`mixed`와 비교하기 쉽게 naming 유지
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
- `flow finished count`
- `total packets`
- 최종 요약:
  - `New / Rtx / RTS / Bounced / ACKs / NACKs / Pulls`
- `tail gap`
  - 예: 알고리즘별 `latest finish` 차이
- 알고리즘별 stability
  - 동일 seed/동일 matrix 반복 시 지표 변동 폭

권장 비교 표:
- baseline(`bitmap`, `mixed`, `oblivious`) 대비
- RTT-aware(`rtt_bitmap`, 이후 `rtt_mixed`) 상대 개선/악화

---

## 8) `UecMpRttBitmap` v0.1 구현 반영

### 8-1. 구현 범위 요약

- `UecMpRttBitmap`은 `UecMpBitmap`과 별도 class로 구현
- `UecMultipath`를 직접 상속
- command-line option: `-load_balancing_algo rtt_bitmap`
- 기존 알고리즘(`bitmap`, `mixed`, `reps`, `reps_legacy`, `oblivious`) 코드는 수정하지 않음

### 8-2. v0.1 실제 정책

- `PATH_GOOD`
  - `raw_rtt > 0`(valid sample)일 때 `min_rtt/srtt/last_valid_rtt`만 업데이트
  - penalty는 항상 `0` (RTT extra penalty 없음)
- `PATH_ECN`
  - base penalty `+1` (기존 Bitmap과 동일)
  - valid RTT + 기존 `srtt`가 있을 때만 extra penalty 최대 `+1`
- `PATH_NACK`
  - base penalty `+4` (기존 Bitmap과 동일)
  - `raw_rtt`는 `srtt` 업데이트에 사용하지 않음
- `PATH_TIMEOUT`
  - base penalty `+_max_penalty` (기존 Bitmap과 동일)
  - `raw_rtt/timeInf`는 RTT sample로 사용하지 않음
- `raw_rtt == 0`
  - invalid/no sample로 처리
  - RTT 상태 업데이트에 사용하지 않음

### 8-3. overflow-safe 비교식

- 기존 위험 비교식 `raw_rtt * 4 > srtt * 5`는 사용하지 않음
- 의미적으로 `raw_rtt > prev_srtt + prev_srtt / 4` 조건을 안전하게 구현
- 현재 구현식:
  - `(raw_rtt > prev_srtt) && ((raw_rtt - prev_srtt) > prev_srtt / 4)`

### 8-4. 실험 결과 요약

#### `one.cm` (`rtt_bitmap`)

| 항목 | 값 |
|---|---|
| 성공 여부 | 성공 |
| flow finished | 176.2 |
| summary | New=490, Rtx=0, RTS=0, Bounced=0, ACKs=124, NACKs=0, Pulls=0 |

#### `perm_16n_16c_2MB.cm` (`rtt_bitmap v0.1`)

| 항목 | 값 |
|---|---|
| 성공 여부 | 성공 |
| flow 수 | 16 |
| earliest finish | 190.066 |
| latest finish | 199.413 |
| total packets | 7840 |
| summary | New=7840, Rtx=0, RTS=0, Bounced=0, ACKs=2877, NACKs=0, Pulls=0 |

#### `perm_16n_16c_2MB.cm` baseline `bitmap` 대비

| 항목 | bitmap baseline | rtt_bitmap v0.1 | 변화 |
|---|---:|---:|---:|
| earliest finish | 187.015 | 190.066 | +3.051 (느림) |
| latest finish | 198.55 | 199.413 | +0.863 (느림) |
| Rtx | 0 | 0 | 동일 |
| NACKs | 0 | 0 | 동일 |
| ACKs | 2775 | 2877 | +102 |

#### `perm_random_1024n_1024c_seed42_0u_16777216b.cm` 결과

| 알고리즘 | New | Rtx | ACKs | NACKs | flow finished count |
|---|---:|---:|---:|---:|---:|
| bitmap | 2833635 | 205 | 1025592 | 205 | 0 |
| rtt_bitmap v0 (참고) | 2823753 | 275 | 1035236 | 275 | 0 |
| rtt_bitmap v0.1 | 2796479 | 584 | 1054893 | 584 | 0 |

참고:
- 해당 random1024 실행 로그에서는 `Flow ... finished at ...` 라인이 없어 earliest/latest 비교는 불가.
- `flow finished count`가 0으로 관찰되어 `-end` 설정 또는 finished log 출력 조건 점검이 필요함.

### 8-5. 결론 및 다음 후보

- `rtt_bitmap v0.1`은 기능적으로 동작하며 작은 workload(`one.cm`, `perm16`)에서 실행 안정성은 확보됨.
- `perm16`에서는 `Rtx/NACK` 안정성은 baseline과 동일하지만 `latest finish`가 baseline `bitmap`보다 여전히 느림.
- random1024에서는 `Rtx/NACK/ACK`가 증가하여 현재 정책이 개선으로 이어지지 않음.
- 따라서 현재 `rtt_bitmap`은 **experimental baseline**이며 최종 개선안은 아님.

다음 튜닝 후보:
1. RTT penalty를 `PATH_ECN`에서도 더 약하게 하거나 비활성화
2. RTT를 penalty가 아니라 path ordering tie-breaker로 사용
3. `RTT-Mixed` 구현/평가로 이동
