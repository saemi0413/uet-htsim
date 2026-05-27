# UEC Multipath 코드워크

이 문서는 `htsim/sim/uec_mp.h`와 `htsim/sim/uec_mp.cpp`에 있는 기존 UEC
multipath 구현을 설명한다. 설명용 문서이며 simulator behavior는 변경하지
않는다.

핵심 관점은 이 클래스들이 물리 링크를 직접 고르지 않는다는 점이다. 이들은
패킷의 `path_id`로 실리는 entropy 값을 고른다. 이후 topology/routing 계층이
그 entropy 값을 실제 사용 가능한 path 중 하나로 매핑한다.

## 1. `UecMultipath` 공통 인터페이스

`UecMultipath`는 모든 UEC multipath 정책의 추상 base class다.

- `PathFeedback`은 ACK, NACK, timeout 처리에서 sender가 특정 entropy/path에
  대해 알게 된 feedback을 나타낸다.
- `EvDefaults::UNKNOWN_EV`는 feedback은 있지만 원래 entropy 값을 모를 때
  사용한다. 현재는 retransmission timeout에서 이 값이 쓰인다.
- 생성자는 `_debug`를 저장하고 `_debug_tag`를 빈 문자열로 초기화한다.
- `set_debug_tag()`는 debug log에 사용할 printable tag를 붙이는 hook이다.
- `processEv(path_id, feedback)`은 feedback hook이다. 구체 정책은 이 함수에서
  path 상태를 갱신한다.
- `nextEntropy(seq_sent, cur_cwnd_in_pkts)`는 send-side hook이다. 구체 정책은
  다음 outgoing packet에 쓸 entropy 값을 반환한다.

`nextEntropy()`의 두 인자는 정책에 넘겨주는 context다.

- `seq_sent`: 전송하려는 sequence number.
- `cur_cwnd_in_pkts`: 현재 congestion window를 packet 단위로 표현한 값.

현재 구현 대부분은 둘 중 하나 또는 둘 다 사용하지 않는다. `UecMpRepsLegacy`는
첫 window를 다르게 처리하기 위해 두 값을 사용한다.

## 2. `PathFeedback` 의미

`PATH_GOOD`은 sender가 해당 entropy에 대해 긍정적인 feedback을 받았다는
뜻이다. 현재 call site 기준으로는 ECN echo가 없는 ACK, 또는 last-hop
trim/NACK 처리 중 성공적인 path feedback으로 취급되는 경우에서 온다.

`PATH_ECN`은 ACK 계열 신호를 받았지만 ECN echo가 설정되어 있다는 뜻이다. loss
신호는 아니지만 해당 path에 congestion이 있었음을 나타낸다.

`PATH_NACK`은 sender가 last hop이 아닌 곳에서 온 NACK/trim 신호를 받았다는
뜻이다. Bitmap은 이를 ECN보다 강한 negative feedback으로 취급한다.

`PATH_TIMEOUT`은 retransmission timer가 만료되었다는 뜻이다. 현재 sender는
packet별 원래 entropy를 저장하지 않으므로 timeout feedback은 `UNKNOWN_EV`와
함께 전달된다.

## 3. `UecMpOblivious`

`UecMpOblivious`는 baseline uniform spraying 정책이다. 낮은 entropy bit를
순회하고, 각 cycle마다 random XOR 값으로 순서를 섞는다.

Header state:

- `_no_of_paths`: entropy bucket 수. 구현은 이 값이 2의 거듭제곱이라고 가정한다.
- `_path_random`: 반환되는 모든 entropy에 더해지는 고정 random upper bits.
- `_path_xor`: 현재 cycle에서 low bits를 permute하는 random 값.
- `_current_ev_index`: XOR permutation 전의 다음 low-bit index.

Constructor, lines 7-24:

- Lines 7-11: base class를 초기화하고 `_no_of_paths`를 저장하며
  `_current_ev_index`를 0으로 시작한다.
- Line 14: entropy 값의 random upper bits를 고른다. 이 upper bits는 policy
  object 생애 동안 고정된다.
- Line 15: path 범위 안에서 첫 XOR permutation 값을 고른다.
- Lines 17-23: debug가 켜져 있으면 constructor state를 출력한다.

`processEv()`, lines 26-28:

- 모든 feedback을 무시한다. 이것이 oblivious spraying behavior를 유지한다.

`nextEntropy()`, lines 30-44:

- Line 32: `mask = _no_of_paths - 1`을 만든다. 이 값은 `_no_of_paths`가 2의
  거듭제곱일 때만 path-index mask로 올바르게 동작한다.
- Line 33: 현재 sequential index와 `_path_xor`를 XOR한 뒤 mask를 적용해 low
  entropy bits를 계산한다.
- Lines 36-40: `_current_ev_index`를 증가시킨다. cycle이 `_no_of_paths`에
  도달하면 index를 0으로 wrap하고 새 `_path_xor`를 고른다.
- Line 42: `_path_random`에서 low bits를 지우고 남은 upper bits를 `entropy`에
  OR한다.
- Line 43: 최종 entropy 값을 반환한다.

결과적으로 각 cycle은 모든 low-bit path ID를 한 번씩 커버하지만 순서는
randomized된다. Feedback은 이 sequence를 바꾸지 않는다.

## 4. `UecMpBitmap`

`UecMpBitmap`은 oblivious 방식의 순회에 path별 skip penalty를 더한 정책이다.
Negative feedback은 path의 penalty를 증가시킨다. 이후 send에서는 penalized
path를 건너뛰려고 시도하면서, 처음 만난 penalized path의 penalty를 점진적으로
감소시킨다.

Header state:

- `_no_of_paths`, `_path_random`, `_path_xor`, `_current_ev_index`는
  `UecMpOblivious`와 같은 역할이다.
- `_ev_skip_bitmap[path]`는 low-bit path의 현재 skip penalty를 저장한다.
- `_ev_skip_count`는 현재 nonzero penalty를 가진 path 수를 추적한다. 이 파일
  안에서는 갱신되지만 selection에는 별도로 사용되지 않는다.
- `_max_penalty`는 path별 penalty 상한이다.

Constructor, lines 47-73:

- Lines 47-52: 공통 state를 초기화하고 skip bitmap은 비어 있는 상태,
  skip count는 0으로 시작한다.
- Line 55: `_max_penalty`를 15로 설정한다.
- Lines 57-58: 고정 random upper bits와 초기 XOR permutation 값을 고른다.
- Lines 60-63: `_ev_skip_bitmap` 크기를 `_no_of_paths`로 맞추고 모든 entry를
  0으로 초기화한다.
- Lines 65-72: debug가 켜져 있으면 constructor state를 출력한다.

`processEv()`, lines 75-96:

- Line 77: low-bit mask를 만든다.
- Line 78: `path_id`를 bitmap index로 사용할 low bits만 남긴다.
- Lines 80-81: feedback이 `PATH_GOOD`이 아니고 이 path가 이전에 penalized
  상태가 아니었다면 `_ev_skip_count`를 증가시킨다.
- Line 83: penalty를 0으로 시작한다.
- Lines 85-90: feedback을 penalty로 매핑한다. ECN은 1, NACK은 4, timeout은
  `_max_penalty`, good feedback은 0을 더한다.
- Line 92: 선택된 path의 skip count에 penalty를 더한다.
- Lines 93-95: 해당 skip count를 `_max_penalty`로 clamp한다.

중요한 edge case: timeout feedback은 현재 `UNKNOWN_EV`와 함께 전달되며, 그 값은
0이다. Bitmap은 `path_id`를 mask하므로 timeout feedback은 실제 timed-out path가
아니라 low-bit path 0을 penalize한다.

`nextEntropy()`, lines 98-135:

- Line 100: low-bit mask를 만든다.
- Line 101: Oblivious와 같은 index-XOR-mask pattern으로 candidate low-bit
  entropy를 계산한다.
- Lines 102-103: loop state를 초기화한다. `flag`는 이 호출에서 이미 하나의
  penalty를 감소시켰는지 기록하고, `counter`는 무한 탐색을 막는다.
- Line 104: 현재 candidate의 skip penalty가 양수인 동안 loop를 돈다.
- Lines 105-111: 이 호출에서 처음 만난 penalized candidate에 대해서만 penalty를
  감소시킨다. 감소 결과가 0이 되면 `_ev_skip_count`도 감소시킨다.
- Lines 113-117: penalized candidate를 하나 처리했음을 표시하고 loop counter를
  증가시키며, `_no_of_paths`보다 많이 scan하면 break한다.
- Lines 118-123: cyclic index를 증가시키고, 필요하면 wrap 및 `_path_xor`
  refresh를 수행한 뒤 다음 candidate를 계산한다.
- Lines 127-131: 다음 호출을 위해 한 번 더 index를 증가시키며 같은 wrap 동작을
  수행한다.
- Line 133: 고정 random upper bits를 OR한다.
- Line 134: 최종 entropy를 반환한다.

결과적으로 feedback은 일시적인 skip pressure를 만든다. 모든 path가 penalized
상태이거나 scan limit에 걸리면 candidate를 그대로 반환하는 fallback behavior가
있다.

## 5. `UecMpReps`

`UecMpReps`는 newer REPS 구현이다. `buffer_reps.h`의
`CircularBufferREPS<uint16_t>`를 사용해 재사용 가능한 good entropy를 저장하고
fresh/frozen behavior를 처리한다.

Header state:

- `_no_of_paths`: entropy 선택지 수.
- `circular_buffer_reps`: policy가 사용하는 heap-allocated circular buffer.
- `_crt_path`: random fallback 시 사용하는 last/current path 변수.
- `_next_pathid`: 여기 선언되어 있지만 현재 구현에서는 사용되지 않는다.
- `_is_trimming_enabled`: timeout-triggered frozen mode를 허용할지 제어한다.

Constructor, lines 137-150:

- Lines 137-141: base class, path count, current path, trimming flag를 초기화한다.
- Line 143: default REPS buffer size로 `CircularBufferREPS`를 allocate한다.
- Lines 145-149: debug가 켜져 있으면 constructor state를 출력한다.

`processEv()`, lines 152-175:

- Lines 154-162: timeout feedback을 처리한다. policy가 아직 frozen 상태가 아니고
  exploration period도 아니라면, trimming이 켜져 있을 때 timeout으로 frozen mode에
  들어간다.
- Line 156: frozen mode를 켠다.
- Line 157: frozen mode를 빠져나갈 수 있는 시간을 기록한다. 값은 현재 simulator
  time에 `exit_freeze_after`를 더한 것이다.
- Lines 158-161: trimming이 꺼져 있으면 abort한다. 이 구현은 trimming 없이
  frozen mode에 들어가는 동작을 지원하지 않는다.
- Lines 164-168: 설정된 시간이 지나면 frozen mode를 끄고 buffer를 reset한 뒤
  `explore_counter`를 16으로 설정한다.
- Lines 170-174: `PATH_GOOD`이면 `path_id`를 circular buffer에 추가한다. 현재 두
  branch는 frozen 여부와 무관하게 모두 `add(path_id)`를 호출한다.

`nextEntropy()`, lines 177-196:

- Lines 178-181: exploration을 구현한다. `explore_counter`가 양수인 동안 counter를
  감소시키고 random path를 반환한다.
- Lines 183-188: frozen mode를 처리한다. buffer가 비어 있으면 random 선택을 하고,
  아니면 `remove_frozen()`으로 entropy를 꺼낸다.
- Lines 189-195: normal mode를 처리한다. buffer가 비어 있거나 fresh entropy가
  없으면 random path를 선택해 `_crt_path`에 저장하고, 아니면 earliest fresh
  entropy를 꺼낸다.

결과적으로 REPS는 good feedback에서 학습해 good entropy를 recycle하며, newer
circular-buffer 구현에서는 timeout-triggered frozen/exploration mechanism을 가진다.

## 6. `UecMpRepsLegacy`

`UecMpRepsLegacy`는 older REPS 구현이다. good entropy의 FIFO list를 유지하고,
random selection으로 fallback하기 전에 이를 recycle한다.

Header state:

- `_no_of_paths`: entropy 선택지 수.
- `_crt_path`: policy가 반환하는 current path 값.
- `_next_pathid`: feedback에서 학습한 good path ID의 FIFO list.

Constructor, lines 199-209:

- Lines 199-202: base class, path count, current path를 초기화한다.
- Lines 204-208: debug가 켜져 있으면 constructor state를 출력한다.

`processEv()`, lines 211-218:

- Line 212: `PATH_GOOD`인지 확인한다.
- Line 213: good `path_id`를 `_next_pathid`에 append한다.
- Lines 214-216: 추가한 path와 FIFO size를 debug log로 출력한다.
- good이 아닌 feedback은 무시한다.

`nextEntropy()`, lines 220-248:

- Line 221: flow가 아직 첫 sending window 안에 있는지 확인한다.
  조건은 `seq_sent < min(cur_cwnd_in_pkts, _no_of_paths)`이다.
- Lines 222-225: `_crt_path`를 round-robin으로 증가시키고 `_no_of_paths`에서
  wrap한다.
- Lines 227-228: first-window debug output을 출력한다.
- Lines 230-245: 첫 window 이후 steady state를 처리한다.
- Lines 231-237: good-path FIFO가 비어 있으면 random path를 선택한다.
- Lines 238-244: FIFO entry가 있으면 front를 pop해 해당 good path를 recycle한다.
- Line 247: `_crt_path`를 반환한다.

`nextEntropyRecycle()`, lines 250-261:

- Lines 251-252: recycle할 good path가 없으면 빈 `optional`을 반환한다.
- Lines 254-255: front good path를 pop해서 `_crt_path`에 저장한다.
- Lines 257-258: MIXED label로 debug output을 출력한다.
- Line 259: recycled path를 반환한다.

결과적으로 first-window traffic은 round-robin이고, 이후 traffic은 ACK된 good
path를 우선 사용하며, recycled path가 없을 때 random으로 fallback한다.

## 7. `UecMpMixed`

`UecMpMixed`는 `UecMpRepsLegacy`와 `UecMpBitmap`을 조합한다. 먼저 REPS 방식의
good-path recycling을 시도하고, 없으면 Bitmap을 fallback으로 사용한다.

Header state:

- `_bitmap`: Bitmap policy instance.
- `_reps_legacy`: legacy REPS policy instance.

Constructor, lines 264-269:

- Lines 264-268: base class를 초기화하고 같은 path count와 debug flag로 두
  sub-policy를 생성한다.

`set_debug_tag()`, lines 271-274:

- mixed policy는 debug tag를 두 sub-policy에 모두 전달한다.

`processEv()`, lines 276-279:

- Line 277: 모든 feedback event를 Bitmap에 전달하므로 negative feedback이 skip
  penalty를 갱신할 수 있다.
- Line 278: 모든 feedback event를 legacy REPS에도 전달한다. legacy REPS는 이 중
  `PATH_GOOD` entry만 저장한다.

`nextEntropy()`, lines 281-288:

- Line 282: legacy REPS에 recycled good path만 요청한다.
- Lines 283-284: recycled value가 있으면 그 값을 반환한다.
- Lines 285-286: good path가 없으면 Bitmap selection으로 fallback한다.

결과적으로 Mixed는 알려진 good entropy reuse를 우선하고, good-path FIFO가 비어
있을 때 Bitmap의 penalty-based avoidance를 사용한다.

## 8. `nextEntropy()` 호출 위치 요약

UEC sender는 `htsim/sim/uec.cpp`에서 outgoing packet의 `path_id`에 값을 쓰기 전에
`_mp->nextEntropy()`를 호출한다.

- `UecSrc::sendPacket()`에서 data packet은 `ev =
  _mp->nextEntropy(_highest_sent, (uint64_t)_cwnd/_mss)`를 통해 entropy를 받고,
  이후 `p->set_pathid(ev)`를 호출한다.
- `UecSrc::sendRtxPacket()`에서 retransmitted data packet도 같은 방식으로 fresh
  entropy를 받는다. 인자는 `_highest_sent`와 현재 cwnd packet 수다.
- `UecSrc::sendProbe()`에서 probe packet도 `_mp`에서 entropy를 받는다.
- `UecSrc::sendRTS()`에서 RTS control packet도 `_mp`에서 entropy를 받는다.

별도로 `UecSink::nextEntropy()`가 pull packet용으로 존재하지만, 이 문서에서
설명하는 `UecMultipath` object와는 별개다.

## 9. `processEv()` 호출 위치 요약

UEC sender는 path feedback을 받을 때 `_mp->processEv()`를 호출한다.

- ACK 처리에서는 `pkt.ev()`를 `PATH_GOOD` 또는 `PATH_ECN`으로 보고한다. 둘 중
  어느 값인지는 `pkt.ecn_echo()`에 따라 결정된다.
- NACK/trim 처리에서는 `pkt.last_hop()`이 true이면 event를 ECN echo 여부에 따라
  `PATH_GOOD` 또는 `PATH_ECN`으로 취급한다. 그렇지 않으면 `PATH_NACK`으로
  보고한다.
- Retransmission timeout 처리에서는 `processEv(UNKNOWN_EV, PATH_TIMEOUT)`을
  호출한다. 근처 코드 주석에 따르면 sender가 packet별 entropy 값을 저장하지
  않는 한 original timed-out entropy를 복구할 수 없다.

## 10. RTT-based multipath를 만들 때 현재 인터페이스에서 부족한 점

현재 인터페이스는 coarse ACK/NACK/ECN/timeout 기반 정책에는 충분하지만,
RTT-based policy가 일반적으로 필요로 하는 여러 정보를 제공하지 않는다.

- `processEv()`는 `path_id`와 categorical feedback enum만 받는다. measured RTT,
  raw RTT, smoothed RTT, base RTT, queueing delay, ACK receive timestamp를 받지
  않는다.
- Timeout feedback은 original entropy를 잃어버린다. sender가 send record에
  entropy를 저장하지 않기 때문이다. RTT policy는 성공과 실패 양쪽 모두에서
  packet별 path attribution이 필요하다.
- `nextEntropy()`는 sequence number와 cwnd만 받는다. packet type, packet size,
  retransmission count, send time, flow age, current in-flight bytes는 받지 않는다.
- "time T에 entropy X로 packet을 보냈다"를 알려주는 명시적 callback이 없다.
  policy는 send request와 나중의 feedback만 본다.
- Feedback은 physical route나 hop이 아니라 entropy 값 단위다. 이는 현재 설계와
  맞지만, RTT policy를 만들 때 entropy가 직접적인 link identifier가 아니라
  routing input이라는 점을 기억해야 한다.
- 추가 context 없이는 ACK가 original send에 대한 것인지 retransmission에 대한
  것인지 policy가 구분할 수 없다.
- path별 selected count, RTT sample, feedback counter, policy state snapshot을
  위한 표준 instrumentation hook이 없다.

보수적인 RTT-based 확장은 기존 policy를 위해 이 인터페이스를 유지하면서, RTT
sample을 전달하고 packet별 sent entropy를 저장하는 opt-in policy/interface 경로를
추가하는 방식이 될 가능성이 높다.
