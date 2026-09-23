# ittage review at ff68f3c5a

- 실행일: 2026-09-23
- 감사 범위: `requirements.v2.md`의 accepted 요구사항 2개 (R-ITTAGE-5 liveness, R-ITTAGE-6 isolation)
- 실행 구성: claim 9개 → Refuter 9개 병렬 → 원시 finding 12개 → 메커니즘별 병합 5개 → Verifier 6개 독립 실행
- 원시 산출물: `claims.json`, `refute/*.json`, `findings.json`, `verify/*.json`, `verdicts.json`

## Triage

| id | severity | requirement | one-line summary | file:line | accept / reject / intentional |
|---|---|---|---|---|---|
| NF-1 | liveness | R-ITTAGE-5 | 매 cycle 예측 요청이 있으면 useful reset sweep이 한 set도 진행되지 않음 | IttageTable.scala:182 | |
| NF-2 | functional | R-ITTAGE-5 | sweep의 비트마스크가 useful 비트가 아니라 padding 비트를 가리켜 useful이 영원히 지워지지 않음 | Bundles.scala:47-48, IttageTable.scala:179 | |
| NF-3 | isolation | R-ITTAGE-6 | usefulCntValid=0인 update(alt-provider 감점)가 useful 비트를 DontCare 값으로 덮어씀 | IttageTable.scala:178, Ittage.scala:181 | |
| NF-4 | functional | R-ITTAGE-5, R-ITTAGE-6 | write buffer에서 같은 set hit과 drain이 같은 cycle에 겹치면 새 쓰기가 needWrite=0으로 남아 영원히 SRAM에 가지 않음 | WriteBuffer.scala:176, 227 | |
| NF-5 | isolation | R-ITTAGE-6 | write buffer가 같은 set의 대기 쓰기를 통째로 교체해 부분 쓰기(update vs useful clear) 중 하나를 버림 | WriteBuffer.scala:131-133, 175-177 | |

## Confirmed findings

### NF-1 (liveness) · 원시 F-ITTAGE-5-2-a · Verifier CONFIRMED
- 무엇이 깨지나: needReset이 선 뒤에도 sweep 카운터가 진행되지 않아 useful 비트가 전 테이블에서 유지됨. allocate 후보가 고갈됨
- 어떤 입력에서: BPU s1이 매 cycle fire하는 부하. `s1_isIndirect = true.B`가 하드와이어(Ittage.scala:85)라 블록에 indirect가 없어도 모든 테이블에 `io.req.fire`가 매 cycle 1
- 어디서: `usefulCanReset = !(io.req.fire || io.update.valid) && needReset` (IttageTable.scala:182). 진행 조건이 idle cycle뿐. 한 번의 sweep에 테이블당 128 또는 256 idle cycle 필요
- 어떻게 보나: `needReset`=1, `resetSet` 고정, `usefulCanReset`=0이 긴 구간 지속. `ittage_us_tick_reset`이 반복 펄스하는데 `resetFinish` 없음. `ittage_reset_u`가 slice 대비 full 벤치에서 급감
- 수정 방향: sweep이 read와 무관하게 진행되도록 조건에서 `io.req.fire`를 제거하고 write port 충돌만 회피. 설계 판단 필요 (PR draft)

### NF-2 (functional) · 원시 F-ITTAGE-5-5-a, F-ITTAGE-5-3-a, F-ITTAGE-6-2-a(일부) · Verifier CONFIRMED
- 무엇이 깨지나: sweep 쓰기가 SRAM에 도달해도 useful 비트는 0이 되지 않음. reset이 사실상 무효
- 어떤 입력에서: 모든 sweep 쓰기. `UsefulCntWidth = 1`
- 어디서: `IttageEntry`가 `usefulCnt` 다음에 `paddingBit`를 선언(Bundles.scala:47-48). Chisel `asUInt`는 마지막 필드를 최하위 비트에 두므로 word bit0 = paddingBit, bit1 = usefulCnt. `updateUsBitmask = (idx < UsefulCntWidth)` (IttageTable.scala:179)는 bit0만 선택. 마스크는 WriteBuffer와 SRAMTemplate를 1:1로 통과
- 어떻게 보나: sweep cycle의 `writePort.bits.bitmask`(IttageTable.scala:215)가 0x1. reset pass 후에도 `io.resp.bits.usefulCnt`가 1 유지
- 수정 방향: 비트마스크를 고정 인덱스가 아니라 entry 필드 위치에서 유도
- 독립 발견: Refuter 3개(5-3, 5-5, 6-2)가 서로 모른 채 같은 지점에 도달

### NF-3 (isolation) · 원시 F-ITTAGE-5-5-b, F-ITTAGE-6-2-a(일부) · Verifier CONFIRMED
- 무엇이 깨지나: useful 비트를 건드리지 않아야 할 update가 useful 비트를 정의되지 않은 값으로 덮어씀
- 어떤 입력에서: provider가 자신 없어 alt 예측을 썼다가 틀린 경우의 alt-provider 감점 update (`usefulCntValid = 0`, Ittage.scala:338-345). `io.update.usefulCnt`는 이 경로에서 DontCare(Ittage.scala:181)
- 어디서: `updateNoUsBitmask = (idx >= UsefulCntWidth)` (IttageTable.scala:178)가 bit1(usefulCnt)을 포함. NF-2와 같은 레이아웃 원인
- 어떻게 보나: alt update cycle의 bitmask가 `...FFFE`. 이후 그 entry의 useful 값이 학습 이력과 무관하게 바뀜
- 수정 방향: NF-2와 동일

### NF-4 (functional) · 원시 F-ITTAGE-6-3-a, F-ITTAGE-5-4-a, F-ITTAGE-5-3-b · Verifier CONFIRMED
- 무엇이 깨지나: 결정된 쓰기 하나가 write buffer 안에 needWrite=0 상태로 남아 SRAM에 가지 않음. sweep의 useful clear 또는 update가 소실
- 어떤 입력에서: cycle N에 set S로 update 쓰기가 buffer에 대기, cycle N+1에 sweep이 resetSet=S로 같은 buffer에 씀 (역순도 동일). sweep cycle은 정의상 read가 없어 drain이 항상 같은 cycle에 일어남. 버퍼가 가득 차 PLRU victim이 drain 인덱스와 같은 경우도 동일
- 어디서: `IttageWriteReq`에 tag가 없어 setIdx만으로 hit(WriteBuffer.scala:131-133). hit 경로가 entry를 통째로 교체하지만 needWrite를 다시 세우지 않음(:175-177). 같은 cycle drain이 `needWrite := false`를 나중에 연결(:227)
- 어떻게 보나: `WriteBuffer_ittageTable{t}_bank{b}_port0_hit_not_written` 카운터가 needReset 구간에 증가. 기존 `ittage_table_update_drop`은 이 경우를 세지 않음(update.valid로 게이트). 전용 카운터 추가 권장
- 수정 방향: hit 시 needWrite 재설정 또는 hit과 drain의 동시 발생 배제. 공용 모듈(bpu/WriteBuffer.scala) 수정이라 tag 없는 다른 사용처에도 영향. 블록 경로 밖 수정임을 PR에 명시

### NF-5 (isolation) · 원시 F-ITTAGE-6-3-b, F-ITTAGE-5-4-b, F-ITTAGE-5-3-c · Verifier CONFIRMED
- 무엇이 깨지나: 비트마스크가 서로 다른 두 부분 쓰기(all-but-useful vs useful-only)가 병합되지 않고 나중 것이 앞 것을 통째로 교체. 한 쓰기의 필드 갱신이 사라짐
- 어떤 입력에서: sweep 쓰기가 대기 중(같은 bank read로 drain 지연)일 때 같은 set에 alt-provider update가 도착, 또는 그 역순
- 어디서: WriteBuffer.scala:175-177 (`entries := io.write.bits`, bitmask 포함 교체)
- 어떻게 보나: 위 hit_not_written 카운터. reset pass 후 특정 set의 useful이 1로 남음
- 수정 방향: 같은 set hit 시 bitmask OR 병합과 마스크별 데이터 병합

## Uncertain
- 없음

## Coverage

| requirement | claims | confirmed | no-counterexample | rejected | not covered |
|---|---|---|---|---|---|
| R-ITTAGE-5 | 6 (5-1 … 5-6) | 4 claims → NF-1, NF-2, NF-3, NF-4 | 5-1 (reset 펄스는 바쁜 cycle에도 모든 테이블에 전달) | 5-6 | 0 |
| R-ITTAGE-6 | 3 (6-1 … 6-3) | 2 claims → NF-3, NF-4, NF-5 | 6-1 (bank·set 주소 격리 성립) | 0 | 0 |
| R-ITTAGE-1 … 4, 7 | 0 | | | | 아직 proposed 상태. 설계자 리뷰 후 감사 |

## Checked and rejected
- F-ITTAGE-5-6-a: sweep 진행 중 두 번째 reset 펄스가 진행 중인 sweep에 흡수됨. Verifier 판정 REJECTED. 요구사항 R-ITTAGE-5의 문구("무기한 미뤄지거나 잊히지 않는다")는 만족. 매 펄스마다 완전한 sweep을 요구할지는 설계자 판단 (아래 후보 참조)

## Requirement candidates
- [R-ITTAGE-8] useful 초기화 펄스는 각각 모든 set에 대한 완전한 sweep을 유발해야 하며, 진행 중인 sweep에 흡수되어 일부 set이 건너뛰어져서는 안 된다
  - evidence: IttageTable.scala:181-189 (Counter에 restart 없음) | verify/F-ITTAGE-5-6-a.json
  - status: proposed
- [R-ITTAGE-9] tick counter는 빈 자리 없는 할당 시도에서만 증가하고, 할당 성공 시의 감소는 의도된 동작이어야 한다
  - evidence: Ittage.scala:379 `selfUpdate(!allocate.valid)` | docs/ittage_algorithm.md는 증가만 서술 (mas.md open question 2)
  - status: proposed
- [R-ITTAGE-10] write buffer가 가득 차거나 같은 set 충돌로 쓰기를 버릴 때는 반드시 카운터로 관측되어야 한다
  - evidence: IttageTable.scala:276 (update.valid로만 게이트) | WriteBuffer.scala:205
  - status: proposed

## Fix phase
- 실행하지 않음. 로컬 환경에 mill이 없어 `compile` 단계를 수행할 수 없음. triage 후 서버 환경에서 `mode=fix` 실행
