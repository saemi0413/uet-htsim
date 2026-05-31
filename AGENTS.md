# AGENTS.md

## 프로젝트 목표

이 repository는 `ultraethernet/uet-htsim` 기반으로 UEC multipath 동작을 분석하고, 이후 RTT 기반 multipath 알고리즘을 설계/구현/평가하기 위한 작업 공간이다.

주요 목표는 다음과 같다.

1. 기존 UEC multipath 구현을 정확히 이해한다.
2. 기존 baseline behavior를 보존한다.
3. 재현 가능한 baseline 실험을 만든다.
4. 이후 ECN 대신 RTT를 사용하는 multipath policy를 설계한다.
5. 최종적으로 RTT-Bitmap, RTT-REPS, RTT-Mixed 계열 알고리즘을 비교 평가한다.

---

## 작업 환경

작업 환경은 다음을 전제로 한다.

- Windows
- WSL2 Ubuntu
- VS Code
- Codex
- GitHub fork repository

Codex는 명시적으로 요청받은 경우에만 다음 작업을 수행한다.

- 파일 읽기
- 빌드 명령 실행
- simulation 실행
- 작은 code patch 작성
- 문서 생성
- `git diff` 확인

---

## 중요한 파일

UEC multipath 관련 작업을 할 때 우선 확인해야 할 파일은 다음과 같다.

- `htsim/sim/uec_mp.h`
- `htsim/sim/uec_mp.cpp`
- `htsim/sim/uec.h`
- `htsim/sim/uec.cpp`
- `htsim/sim/uecpacket.h`
- `htsim/sim/uecpacket.cpp`
- `htsim/sim/datacenter/main_uec.cpp`
- `htsim/sim/datacenter/fat_tree_switch.*`
- `htsim/sim/datacenter/fat_tree_topology.*`
- `htsim/sim/route.*`

---

## UEC multipath 기본 mental model

`uec_mp.cpp`는 실제 물리 링크를 직접 선택하는 코드가 아니다.

이 파일의 역할은 다음과 같다.

- 다음 packet에 붙일 entropy 또는 `path_id`를 선택한다.
- `UecSrc`가 `UecMultipath::nextEntropy()`를 호출한다.
- 반환된 entropy는 packet에 `set_pathid(...)`로 기록된다.
- ACK / NACK / TIMEOUT 결과는 `UecMultipath::processEv(...)`로 다시 전달된다.

즉 전체 흐름은 다음과 같다.

```text
send packet
  → _mp->nextEntropy(...)
  → packet.set_pathid(ev)
  → packet 전송

feedback 도착
  → ACK / ECN / NACK / TIMEOUT 판정
  → _mp->processEv(path_id, feedback)
  → multipath policy 내부 상태 업데이트