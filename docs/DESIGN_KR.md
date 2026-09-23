# 요구사항 추출·반박 기반 자동 코드 리뷰 Agent 설계 (NLGTM)

- 작성일: 2026-09-23 (2차 개정: 요구사항 추출을 Agent가 수행, 사용자는 리뷰. 언어·도메인 독립)
- 목적: AX 경진대회 제출용 Custom Agent. 코드에서 최소 아키텍처 요약(MAS)과 단순화된 요구사항을 추출하고, 사용자 리뷰를 거친 요구사항을 반례 탐색으로 위반 입력을 찾아 버그를 검출하고 수정 PR까지 생성
- 첫 적용 대상: XiangShan Vanilla (Chisel). 데모는 ITTAGE, mBTB. Agent 자체는 블록·언어에 종속되지 않음
- 제출 형태: GitHub repository, `custom-agent-example/` 하위에 `NLGTM.md`, `skills/`, `configs/`, `examples/`

---

## 1. 배경과 문제

- 팀 구성: 설계자 7명, verification 전담 1명
- Formal / UT 등 verification 인프라 부족. emu 시뮬레이션은 느리고 성능 회귀는 잡아도 기능 버그의 위치는 못 잡음
- 실제로 잡은 버그는 전부 코드 리뷰로 발견 (`docs/Bug/`, `docs/ittage_perf_degradation.md`)
  - pdInvalidate position 규약 불일치 (`b6488b0d5`)
  - pdInvalidate pair second 슬롯 stale meta (`4690ac6a4`)
  - write buffer per-way flush 구분 누락 (`ff68f3c5a`)
  - ITTAGE useful reset sweep이 idle cycle에만 진행 → full bench에서 정지
  - ITTAGE useful 마스크 padding bit (upstream #6569와 동일)
  - backend branch target 검증의 stale 참조 (`7d93eacc3`, targetMem 도입). 2-taken 도입 후 gcc difftest에서 레지스터가 한 iteration 어긋나는 증상. 원인은 예측 target을 후속 entry의 `pcMem[ftqIdx+1]`에서 읽는 구조라 s3_override로 후속 enqueue가 취소되고 FTQ full로 지연되면 이전 세대 값이 검증을 통과함
- 검증 환경의 턴어라운드가 병목. ZeBu 에뮬레이션은 한 턴에 2~3일. targetMem 건은 파형 확보, 가설, 수정, 재실행을 반복하며 한 달 이상 소요. 선행 시도 2건(s3_override 시 pcMem 갱신, predecode target fault)이 실패한 뒤에야 구조적 원인에 도달
- 같은 건을 리뷰 관점에서 보면 "entry 자신의 예측 target과 비교해야 한다"는 단순한 요구사항의 반례(다른 entry의 상태에 의존, 그 상태는 취소·지연 가능)로 코드 추적만으로 도달 가능. 턴어라운드 2~3일이 리뷰 1회로 대체됨
- 리뷰가 유효한 방법임은 입증됐으나 사람 시간이 병목. 리뷰 자체를 자동화 대상으로 삼음
- 스펙을 사람이 쓰는 방식은 유지되지 않음. 단순화된 요구사항조차 사람이 쓰면 안 쓰게 됨 → 추출도 Agent가 하고 사람은 리뷰만

## 2. 핵심 원칙

- **코드 → MAS → 요구사항은 Agent가 만든다.** 사용자는 읽고 accept / edit / reject / add 표시만
- **요구사항은 짧게, 목적으로. 상세 스펙이 아니라 단순화된 요구사항으로 버그를 잡는 것이 핵심.** "Y일 때 X가 반드시 일어나야 한다". 메커니즘 서술 금지. 근거(주석, assert, 카운터 이름, 커밋 제목)를 첨부해 사용자가 몇 초 만에 판단
- **리뷰 모드는 반례 탐색.** "코드가 무엇을 하는가"가 아니라 "이 요구사항을 위반하는 입력이 있는가"
- **Agent 간 대립 구조.** 반례를 찾는 Refuter와 그 반례를 다시 반박하는 Verifier를 분리. 서로의 추론을 보지 않음
- **사람 개입은 3곳.** 요구사항 리뷰(반복), finding triage, PR merge. 그 사이는 무인
- **언어·도메인 독립 코어 + 프로젝트 config 분리.** 코어는 trigger → effect, 가드, 인코딩, 폭, stale, liveness, isolation만 다룸. Chisel, SystemVerilog, 소프트웨어 모두 config 교체로 대응

## 3. 순환 문제와 그 해결

- 위험: 코드에서 뽑은 요구사항은 코드의 가정을 되풀이함. 코드가 스펙이면 비교 대상이 없음
- 해결 1: 요구사항을 **목적** 수준으로만 적음. "useful counter는 주기적으로 reset되어야 한다"는 코드의 sweep 구현과 독립. 구현이 idle에만 진행돼도 목적 문장은 그대로라 반례가 성립
- 해결 2: 근거를 코드 본문이 아닌 **의도 신호**에서 우선 취함. 주석, assert 메시지, perf counter 이름(`pd_invalidate`가 있다는 것 자체가 "invalidate가 일어나야 한다"는 의도), 커밋 제목, 설계 문서
- 해결 3: 사용자 리뷰. 문장이 목적으로 적혀 있으면 설계자는 코드를 열지 않고도 맞고 틀림을 판단. 이 판단이 의도와 구현을 분리하는 지점
- 해결 4: 감사 후 Judge가 "코드에 있는데 요구사항이 없는 메커니즘"을 후보로 제안. 아무도 안 적은 요구사항(예: targetWrong rewrite)을 점진적으로 보충

## 4. 왜 단순화된 요구사항으로 충분한가

- 과거 버그를 요구사항 위반으로 재기술하면 모두 단순한 요구사항의 반례

  | 요구사항 (목적) | 반례 (실제 버그) |
  |---|---|
  | 틀린 것으로 판정된 BTB entry는 invalidate되어야 한다 | bank1 시작 block에서 wayMask = 0 (논리/물리 bank 비교 불일치) |
  | 〃 | pair second 슬롯에서 무관한 entry invalidate, 원인 entry 잔존 |
  | invalidate는 대상 way 외에 영향을 주면 안 된다 | write buffer가 way 구분 없이 flush → 다른 way 손실 |
  | useful counter는 주기적으로 reset되어야 한다 | 연속 prediction 중 sweep 정지 |
  | 〃 | 마스크 폭 < 필드 폭, 일부 way 미reset |
  | branch target 검증은 그 branch가 속한 entry 자신의 예측 target과 비교해야 한다 | `pcMem[ftqIdx+1]` 참조. 후속 entry enqueue가 s3_override로 취소되고 FTQ full로 지연되면 이전 세대 값과 비교 → 실제 mispredict 은폐 |

- 버그는 요구사항의 부재가 아니라 **요구사항이 성립하지 않는 입력 클래스**에서 나옴
- 입력 클래스 분해(어느 bank, 어느 슬롯, idle/busy, SRAM/VC)는 `conventions.md`의 분해 축을 따라 Agent가 자동 수행

## 5. 산출물 형식

### 5.1 MAS (`mas/<block>.md`, 블록당 120줄 이내)

- Purpose: 소비자 관점에서 이 블록이 무엇을 위한 것인가, 1~2문장
- Interfaces: 포트 그룹별 한 줄
- State: 소유한 저장소별 한 줄 (크기, 인덱스, 읽는/쓰는 메커니즘)
- Mechanisms: `trigger → effect` 한 줄씩, 양끝 `file:line`. 설정 게이트 표기
- Intent signals: 주석·assert·카운터·커밋 제목 인용
- Open questions: 목적이 어디에도 적혀 있지 않은 동작

### 5.2 요구사항 (`requirements/<block>.md`, 실행 간 유지되는 유일한 산출물)

```
- [R-ITTAGE-1] useful counter는 tickCnt 포화 시 모든 테이블, 모든 set에 대해 유한 시간 내 reset되어야 한다
  - evidence: IttageTable.scala:181-188 | "ittage_us_tick_reset"
  - observe: ittage_reset_u
  - status: proposed
```

- `status`: proposed(Agent 제안) → accepted / edited / rejected(+why) / owner(사용자 추가)
- accepted 줄은 Agent가 절대 수정하지 않음. 새 줄 제안만
- rejected + "의도된 동작"은 `## intentional` 블록으로 이동 → 반례 탐색에서 제외

## 6. 루프

```
 code + docs + comments + counters + asserts + git log
        │
        v
 [0a extract-mas]            블록 → mas/<block>.md
        │
        v
 [0b extract-requirements]   mas → requirements/<block>.md (status: proposed)
        │
 ── 사용자 리뷰: accept / edit / reject / add ──   ← 0b를 iterate 모드로 반복, stable까지
        │
        v
 [1 decompose-requirement]   accepted 요구사항 → claims.json (요구사항 × 분해 축 × 형제 경로 × 경계 × liveness × isolation)
 [2 refute-claim × N]        claim → finding 또는 NO_COUNTEREXAMPLE        (병렬 서브에이전트)
 [3 verify-finding × N]      finding → CONFIRMED / REJECTED / UNCERTAIN    (병렬, Refuter 추론 미공유)
 [4 judge-and-report]        → report.md (triage 표, 커버리지, 요구사항 후보)
        │
 ── 사용자 triage: accept / reject / intentional ──
        │
 [5 fix-and-compile]         accept finding → 브랜치, 최소 패치, compile, (observe 비교), PR
        │
 ── 사용자 merge ──
```

### 6.1 단계별 정의

| 단계 | 입력 | 출력 | 역할 |
|---|---|---|---|
| 0a extract-mas | 블록 경로, docs, git log | mas.md | 목적·인터페이스·상태·메커니즘·의도 신호 추출 |
| 0b extract-requirements | mas.md, 기존 requirements | requirements.md | 메커니즘·의도 신호별 obligation 도출. iterate 모드에서 리뷰 표시 반영 |
| 1 Decomposer | accepted 요구사항 1줄, conventions | claim 목록 | 요구사항을 입력 클래스·형제 경로·경계·liveness·isolation으로 전개 |
| 2 Refuter | claim 1개 | finding 0~n개 | 가드·인코딩·폭·stale·liveness·isolation 관점으로 위반 입력 구성. 못 찾으면 추적 기록과 함께 NO_COUNTEREXAMPLE |
| 3 Verifier | finding의 결론만 | CONFIRMED / REJECTED / UNCERTAIN | 독립 재추적. 도달 가능성, 다른 경로의 보완, 요구사항 위반 여부 |
| 4 Judge | 전체 | report.md | 병합, 심각도(functional/liveness/isolation/latent), 커버리지, 요구사항 후보 |
| 5 Fixer | accept finding | PR | 최소 패치, compile 필수, observe 카운터 비교, merge 금지 |

### 6.2 사용자 개입

| 게이트 | 무엇을 | 시간 목표 |
|---|---|---|
| 요구사항 리뷰 | 줄마다 status 표시. 필요 시 한 줄 추가 | 블록당 5~10분 × 2~3회 |
| finding triage | report.md 표에서 accept / reject / intentional | 항목당 1분 |
| PR merge | 일반 PR 리뷰 | PR당 수 분 |

- 신뢰가 쌓이면 triage 게이트는 PR 리뷰로 통합 가능. 요구사항 리뷰는 유지 (의도의 유일한 원천)

### 6.3 트리거

- 온디맨드: 블록 최초 extract → 리뷰 → audit
- PR diff: 변경 파일이 속한 블록의 accepted 요구사항만, diff에 닿는 claim만 재실행
- 주기: 주 1회 전체 sweep. upstream diff를 참조 자료로 추가

## 7. 에이전트 역할 정의 원칙

- Refuter와 Verifier는 서로의 출력을 보지 않음. Verifier는 claim, 실패 입력, 위치만 받음
- finding 필수 필드: requirement id, claim id, `file:line`, 실패 입력, 실패 결과, 관측 방법. 하나라도 비면 Judge가 폐기
- 코드 인용은 rev 고정 (`<rev>:<path>:<line>`)
- 기능 위반만 보고. 성능·타이밍 추정 금지
- 설정 게이트: `active_config`에서 도달 불가한 경로는 버그 아님. 대안 설정에서 도달 가능하면 `latent`

## 8. 확장 구조

```
custom-agent-example/
  NLGTM.md                  # Agent 본체. 언어·도메인 독립
  skills/
    extract-mas/SKILL.md
    extract-requirements/SKILL.md
    decompose-requirement/SKILL.md
    refute-claim/SKILL.md
    verify-finding/SKILL.md
    judge-and-report/SKILL.md
    fix-and-compile/SKILL.md
  configs/
    _template/                     # blocks.yaml, conventions.md, requirements/BLOCK.md
    xiangshan/
      blocks.yaml                  # 26개 블록 (frontend/backend/mem/cache), 빌드, 관측, active_config
      conventions.md               # 분해 축, 형제 경로, 인코딩 규약
      requirements/                # Agent 생성, 사용자 리뷰
  examples/
    ittage-useful-reset/
    mbtb-pd-invalidate/
```

- 코어(`NLGTM.md`, `skills/`)에는 블록 이름도 언어 이름도 없음
- 같은 프로젝트 새 블록: `blocks.yaml` 한 항목 + `mode=extract`
- 다른 프로젝트: `configs/_template/` 복사. 분해 축이 "bank / slot"에서 "tenant / retry / partial failure"로 바뀔 뿐 절차 동일

## 9. 데모 시나리오

### 9.1 ITTAGE (Vanilla 레포, 버그 존재 상태)

- extract 기대: Intent signals에 `ittage_us_tick_reset`, `ittage_reset_u` 카운터, `resetUsefulCnt` 포트가 잡힘 → 요구사항 "useful counter는 tickCnt 포화 시 모든 set에 대해 유한 시간 내 reset되어야 한다" 제안
- 사용자 리뷰: accepted
- Decomposer 기대 claim: 트리거 발생, sweep 진행 조건, 전체 set 커버, 필드 전체 폭
- Refuter 기대 finding
  - liveness: `usefulCanReset := !(io.req.fire || io.update.valid) && needReset` (`IttageTable.scala:182`). 매 cycle req 발행 시 진행 0
  - width: 마스크가 padding bit 포함 폭으로 생성되어 실제 필드와 정렬 불일치
- 정답지: upstream #6569

### 9.2 mBTB pdInvalidate (rev `182ce7745`)

- extract 기대: `pd_invalidate` 카운터, `pdFlush` 포트 → "틀린 것으로 판정된 entry는 모든 저장 매체에서 invalidate되어야 한다", "invalidate는 대상 way 외에 영향을 주면 안 된다"
- 기대 finding 3건이 각각 `b6488b0d5`, `4690ac6a4`, `ff68f3c5a`에 대응
- 형제 경로 대칭성(train에는 pair second 가드, invalidate에는 없음)이 Decomposer의 sibling claim에서 나와야 함

### 9.3 backend targetMem (rev `7d93eacc3` 직전, 2-taken 활성)

- extract 기대: CtrlBlock의 branch target 비교 메커니즘 `pcMem read @ ftqIdx+1 → targetWrong → redirect`가 MAS Mechanisms에 잡힘. Open questions에 "예측 target을 왜 다음 entry에서 읽는가"가 남아야 함
- 요구사항 제안: "branch target 검증은 그 branch가 속한 entry 자신의 예측 target과 비교해야 한다"
- Decomposer 기대 claim: stale(참조하는 상태가 다른 entry의 것인가), 경계(후속 entry가 아직 enqueue되지 않은 경우), redirect 출처 축(s3_override 후), 큐 포인터 축(FTQ full)
- Refuter 기대 finding: stale. 실패 입력은 s3_override가 pair를 취소하고 FTQ full로 재enqueue가 지연된 상태에서 entry N의 branch가 실행되는 경우. 관측은 difftest 레지스터 불일치 또는 redirect 없이 진행되는 targetWrong=0
- 정답지: `7d93eacc3`. ZeBu 기준 한 달 이상 걸린 건이 리뷰 1회 범위임을 보이는 예시

## 10. 제출 양식 매핑

1. Agent 이름: NLGTM
2. 타겟 업무: 설계 블록 기능 리뷰 및 버그 수정. 코드에서 요구사항을 추출하고 사용자 리뷰 후 반례 탐색
   - 중요성: 7:1 설계/검증 비율, Formal/UT 등 verification 인프라 부족. ZeBu 한 턴 2~3일, 버그 1건에 한 달 이상 소요 사례. 리뷰는 턴어라운드가 없는 유일한 검출 수단
   - 기존 투입: 블록 감사 1건당 수동 분석 수 세션, 인지에서 수정까지 수일~한 달. 팀 기록으로 수치 확정 필요
3. Repository: `custom-agent-example/` (§8)
4. 실행 예시: §9
- 산출물은 두 개. repository 하나와 Confluence 제출 양식(`custom-agent-example/SUBMISSION_KR.md`)

## 11. 리스크와 미해결

- 추출 요구사항의 순환성: §3의 4중 장치로 완화. 완전 해결 아님. 사용자 리뷰 품질에 의존
- 오탐 비용: Verifier 독립성에 의존. 초기에는 triage 게이트 유지
- Decomposer 품질: 분해 축이 얕으면 반례를 못 찾음. `conventions.md`에 프로젝트별 축을 명시
- Fixer 검증 수단: compile + observe 카운터뿐. merge 전 사람 리뷰 필수
- 실행 환경: 로컬에 mill 없음. compile 단계는 서버 환경 전제

## 12. 다음 단계

1. ITTAGE에 `mode=extract` 실행. `examples/ittage-useful-reset/mas.md`, `requirements.v1.md` 생성
2. 사용자 리뷰 1회 → `iterate` → stable
3. `mode=audit` → report.md. upstream #6569와 대조해 `NOTES_KR.md`
4. mBTB pdInvalidate 동일 절차 (rev `182ce7745`)
5. 제출 양식 문서 작성
