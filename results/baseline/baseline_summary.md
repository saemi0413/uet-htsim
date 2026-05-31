# UEC Multipath Baseline Summary

## 1) Build Command

실행 위치: `htsim/sim`

```bash
cmake -S . -B build
cmake --build build --target htsim_uec --parallel
```

결과: `htsim_uec` target build 성공.

---

## 2) `one.cm` 기본 smoke test 결과

실행 명령:

```bash
./build/datacenter/htsim_uec -tm datacenter/connection_matrices/one.cm -o /tmp/uec_one_logout.dat
```

| 항목 | 값 |
|---|---|
| 성공 여부 | 성공 |
| exit code | 0 |
| flow finished time | 176.2 |
| total packets (flow line) | 490 |
| stdout summary | New=490, Rtx=0, RTS=0, Bounced=0, ACKs=124, NACKs=0, Pulls=0 |
| stderr | 없음 |
| output file | `/tmp/uec_one_logout.dat` 생성 |

---

## 3) `one.cm` 5개 `-load_balancing_algo` smoke test 결과

공통 조건: 단일 flow 실행 검증 (`bitmap`, `reps`, `reps_legacy`, `oblivious`, `mixed`)

| algorithm | 성공 | exit code | earliest finish | latest finish | total packets | New/Rtx/RTS/Bounced/ACKs/NACKs/Pulls | stderr |
|---|---|---:|---:|---:|---:|---|---|
| bitmap | 성공 | 0 | 176.2 | 176.2 | 490 | 490/0/0/0/124/0/0 | 없음 |
| reps | 성공 | 0 | 176.2 | 176.2 | 490 | 490/0/0/0/124/0/0 | 없음 |
| reps_legacy | 성공 | 0 | 176.2 | 176.2 | 490 | 490/0/0/0/124/0/0 | 없음 |
| oblivious | 성공 | 0 | 176.2 | 176.2 | 490 | 490/0/0/0/124/0/0 | 없음 |
| mixed | 성공 | 0 | 176.2 | 176.2 | 490 | 490/0/0/0/124/0/0 | 없음 |

관찰: 단일 flow에서는 알고리즘 간 차이가 거의 드러나지 않음.

---

## 4) `perm_16n_16c_2MB.cm` baseline 결과

| algorithm | 성공 | exit code | flow 수 | earliest finish | latest finish | total packets | New/Rtx/RTS/Bounced/ACKs/NACKs/Pulls | stderr |
|---|---|---:|---:|---:|---:|---:|---|---|
| bitmap | 성공 | 0 | 16 | 187.015 | 198.55 | 7840 | 7840/0/0/0/2775/0/0 | 없음 |
| reps | 성공 | 0 | 16 | 190.095 | 200.611 | 7840 | 7840/0/0/0/2853/0/0 | 없음 |
| reps_legacy | 성공 | 0 | 16 | 192.563 | 204.128 | 7840 | 7840/0/0/0/3003/0/0 | 없음 |
| oblivious | 성공 | 0 | 16 | 189.607 | 210.643 | 7840 | 7840/3/0/0/3311/3/0 | 없음 |
| mixed | 성공 | 0 | 16 | 187.944 | 198.726 | 7840 | 7840/0/0/0/2661/0/0 | 없음 |

관찰: 이 workload에서는 `oblivious`가 tail(`latest finish`)과 `Rtx/NACK` 측면에서 불리.

---

## 5) `perm_random_1024n_1024c_seed42_0u_16777216b.cm` 결과

### 5-1. 파일 생성/검증

- 파일: `datacenter/connection_matrices/perm_random_1024n_1024c_seed42_0u_16777216b.cm`
- 조건 검증 통과:
  - `Nodes 1024`, `Connections 1024`
  - flow 1024개
  - source/destination 각각 `0..1023` 1회씩
  - self-flow 0개
  - `start 0`, `size 16777216` 전부 일치

### 5-2. 1024-flow baseline 실행 (`bitmap`, `mixed`, `oblivious`)

| algorithm | 성공 | exit code | flow 수(start line) | earliest finish | latest finish | total packets 관찰치 | New/Rtx/RTS/Bounced/ACKs/NACKs/Pulls | stderr |
|---|---|---:|---:|---|---|---:|---|---|
| bitmap | 성공 | 0 | 1024 | N/A | N/A | 2833635 | 2833635/205/0/0/1025592/205/0 | 없음 |
| mixed | 성공 | 0 | 1024 | N/A | N/A | 2835750 | 2835750/43/0/0/1012793/43/0 | 없음 |
| oblivious | 성공 | 0 | 1024 | N/A | N/A | 2654712 | 2654712/10475/0/0/1095466/10478/0 | 없음 |

참고: 해당 실행 로그에서는 `Flow ... finished at ...` 라인이 출력되지 않아 finish-time 기반 비교는 불가.

---

## 6) 알고리즘별 초기 해석

| 알고리즘 | 초기 해석 |
|---|---|
| oblivious | 작은 permutation(16-flow)에서 이미 `Rtx/NACK` 발생, 1024-flow에서는 `Rtx/NACK`가 크게 증가. |
| bitmap | 16-flow에서는 안정적, 1024-flow에서도 `oblivious` 대비 훨씬 낮은 `Rtx/NACK`. |
| mixed | 16-flow tail이 bitmap과 유사하고, 1024-flow에서는 `Rtx/NACK`가 3개 중 가장 낮음. |
| reps / reps_legacy | 16-flow에서는 동작 정상. 이번 1024-flow 핵심 3개 비교에는 미포함. |

---

## 7) RTT 기반 변경 전/후 비교 기준(권장)

| 구분 | 비교 항목 |
|---|---|
| 안정성 | `Rtx`, `NACKs`, (가능하면 timeout 관련 지표) |
| 성능(지연) | `earliest finish`, `latest finish`, 필요 시 p50/p95/p99 FCT |
| 진행량/처리량 | `New`, `ACKs`, 시뮬레이션 완료 시점의 finished flow 수 |
| 재현성 | 동일 seed/동일 CM에서 결과 변동 폭 |
| 실행 건전성 | exit code, stderr error 여부 |

---

## 8) 이후 작업 순서

1. `perm_random_1024...` 실행에서 finish-time line이 나오도록 실행 조건(`-end`, 로그 옵션) 정리.
2. 동일 조건으로 `bitmap/reps/reps_legacy/oblivious/mixed` 5종 재수집.
3. baseline 고정본(명령/seed/CM/파라미터) 문서화.
4. RTT policy 후보(`RTT-Bitmap`, `RTT-REPS`, `RTT-Mixed`) 설계/구현.
5. 같은 workload 세트로 before/after 비교표 작성.
6. tail/재전송 중심으로 최종 알고리즘 선택 및 회귀 검증.

