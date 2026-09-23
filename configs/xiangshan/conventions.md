# XiangShan 규약과 분해 축

Agent의 모든 단계가 공통으로 참조한다. 요구사항이 아니라 "이 설계에서 상태가 어떻게 갈라지는가"를 적는다.

## 1. 분해 축 (Decomposer가 claim을 전개할 때 반드시 고려)

| 축 | 클래스 | 적용 블록 |
|---|---|---|
| fetch block 시작 bank | VA[5] = 0 / 1 (align bank 0/1 시작). 논리 순서와 물리 bank가 뒤집힘 | bpu, mbtb, ubtb, abtb, ftq, ifu |
| 2-taken pair 슬롯 | 일반 / pair first / pair second. second는 BPU 메인 파이프를 거치지 않아 meta가 없음 | bpu, ubtb, ftq, ifu, history |
| 저장 매체 | SRAM / write buffer / victim cache / register file. 각각 별도 invalidate·write 경로 | mbtb, tage, ittage, icache, dcache |
| 부하 | idle / 매 cycle req 발행. "idle에만 진행"은 liveness 위반 후보 | 모든 sweep, reset, replacement, refill 로직 |
| redirect 출처 | backend / IFU predecode / BPU S3 override / 예외 | ftq, bpu, ifu, rob, lsq |
| 접근 종류 | fetch / prefetch / uncache / MMIO | ifu, icache, ftq, dcache, ldst_pipe |
| 페이지 경계 | block이 page를 넘음 / 안 넘음 | ifu, icache, tlb, ldst_pipe |
| 압축 명령 | RVC 반쪽이 block 경계에 걸림 | ifu, icache, ibuffer |
| 큐 포인터 | 빈 / 가득 / wrap (flag bit 반전) | ftq, rob, lsq, ibuffer, sbuffer |
| 벡터/스칼라 | scalar / vector split uop / vector last uop | decode, rename, rob, lsq |
| 예외/인터럽트 | 없음 / 명령 예외 / 비동기 인터럽트 / trigger | rob, ctrlblock, ifu |
| 설정 게이트 | `EnableTwoTaken`, `HasVC` 등 Option/if 게이트. 활성 설정은 `blocks.yaml` | 전체 |

## 2. 형제 경로 (Sibling paths)

같은 상태를 소비하는 경로 목록. 한 경로에 있는 가드가 다른 경로에 없으면 1차 버그 후보.

- FTQ 슬롯 meta 소비자: train(commit), pdInvalidate(IFU redirect), backend redirect, S3 override, perf
- BTB 계열 entry 소비자: predict read, train write, invalidate, replacement touch
- ROB entry 소비자: commit, exception, redirect flush, walk, trace
- LSQ entry 소비자: forward, violation check, commit, flush, uncache
- TLB entry 소비자: lookup, refill, sfence, PTW response

## 3. 인코딩 규약

- BPU position: `CfiPositionWidth` = 상위 1비트(fetch block 내 **논리** bank 순서, S0 rotator 부여) + 하위 `CfiAlignedPositionWidth`비트(align bank 내 halfword offset). 물리 bank 번호 VA[5]와 다르다. 비교 시 양쪽이 같은 규약인지 확인
- 큐 인덱스(`CircularQueuePtr`): value + flag. 순서 비교는 `isBefore`/`isAfter`, `needFlush(redirect)`로만. value만 비교하면 wrap에서 틀림
- 파이프라인 신호: `sN_valid`, `sN_ready`, `sN_fire = valid && ready && !kill`, `sN_kill`. `fire`가 아닌 `valid`로 상태를 갱신하면 stall 중 중복 갱신
- 태그: 각 predictor의 tag 폭과 idx 폭이 SRAM entry 정의와 일치해야 함. 마스크는 padding bit를 제외한 실제 필드 폭
- `Reg` without `RegInit`: 이전 점유자 값이 남음. 소비 전에 valid 또는 동일 슬롯의 갱신 조건이 있는지 확인

## 4. 관측

- perf counter: `XSPerfAccumulate("name", cond)`. 이름은 `blocks.yaml`의 `observe`
- 파형 신호 지정 시 반드시 정의 파일:라인 동반
- difftest mismatch는 기능 위반의 최종 관측. 성능 카운터는 liveness/isolation 위반의 관측

## 5. 빌드

- 컴파일: `mill -i xiangshan.compile`. 로컬에 mill이 없으면 서버에서 실행
- 최소 설정: `CONFIG=TLMinimalConfig`
