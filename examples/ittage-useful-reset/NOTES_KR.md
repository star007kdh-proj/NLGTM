# ittage-useful-reset · 실행 노트와 정답 대조

- 대상: XiangShan Vanilla, rev ff68f3c5a, 블록 ittage (7 파일, 1127줄)
- 실행일: 2026-09-23. Claude Code 서브에이전트로 각 단계 실행. 모든 산출물은 Agent가 쓴 그대로이며 이 노트만 사람이 작성

## 실행 흐름과 소요

| 단계 | 에이전트 수 | 산출물 |
|---|---|---|
| 0a 블록 요약 + 0b 요구사항 추출 | 1 | `mas.md`(89줄, 메커니즘 21개, 의도 신호 25개), `requirements.v1.md`(7개, 모두 proposed) |
| 설계자 리뷰 | 사람 | `requirements.v2.md`. R-5, R-6 accepted (제출자 대행), 나머지 proposed 유지 |
| 1 조건 분해 | 오케스트레이터 | `claims.json` 9개 |
| 2 반례 탐색 | 9 병렬 | `refute/*.json` → 원시 finding 12개, 반례 없음 2개 |
| 3 독립 검증 | 6 병렬 | `verify/*.json` → CONFIRMED 5, REJECTED 1 |
| 4 보고서 | 오케스트레이터 | `report.md`. 메커니즘 병합 후 finding 5개, 요구사항 후보 3개 |
| 5 수정 PR | 미실행 | 로컬에 mill 없음 |

## 정답 대조

| 보고서 finding | 사전에 알려진 정답 | 판정 |
|---|---|---|
| NF-1 sweep이 busy 부하에서 정지 (liveness) | `docs/ittage_perf_degradation.md` §4.1 (수동 리뷰로 발견) | **일치** |
| NF-2 sweep 마스크가 padding 비트를 가리킴 | upstream OpenXiangShan #6569 "derive useful counter masks from entry fields" | **일치** |
| NF-3 non-useful update가 useful 비트를 덮어씀 | upstream #6569 (같은 수정으로 해결) | **일치** |
| NF-4 write buffer hit+drain 동시 발생 시 쓰기 소실 | 없음 | **신규**. 설계자 triage 필요 |
| NF-5 write buffer 같은 set 통째 교체로 부분 쓰기 소실 | 없음 | **신규**. 설계자 triage 필요 |

- 알려진 버그 2건(3 메커니즘)을 모두 찾았고, 알려지지 않은 후보 2건을 추가로 찾았다. NF-4, NF-5는 공용 모듈 `bpu/WriteBuffer.scala`에 있어 ITTAGE 밖의 다른 predictor에도 영향을 줄 수 있다
- NF-2는 서로 다른 claim을 맡은 Refuter 3개(5-3, 5-5, 6-2)가 독립적으로 같은 지점에 도달했다. NF-4, NF-5도 Refuter 3개(5-3, 5-4, 6-3)가 독립 발견했다. 수렴 자체가 신뢰도 근거

## 정직하게 적어 둘 것

- 설계자 리뷰 게이트는 제출자가 대행했다. 블록 owner의 실제 리뷰가 아니다. R-1 … R-4, R-7은 아직 proposed
- Refuter 6-2는 추적 중 `git log`로 upstream 커밋(#6569)을 참조했다. 정답지 누출에 해당하므로 NF-2, NF-3의 근거로는 이를 참조하지 않은 5-5와 5-3의 결과만 사용했다. Verifier 6개는 모두 upstream 참조를 금지한 상태로 실행했다
- Verifier가 Refuter의 입력 한 곳을 정정했다 (테이블별 tagLen이 다르다는 서술 → 실제로는 전 테이블 TagWidth=9). 결론에는 영향 없음. 독립 검증이 오류를 잡는 사례
- F-5-6-a는 Refuter가 반례로 제출했으나 Verifier가 요구사항 문구 기준으로 기각했다. 기각 사유가 "요구사항이 이 경우를 요구하는지 불명확"이라 요구사항 후보 R-8로 되돌렸다. 그림의 "요구사항이 없는 동작 → 새 요구사항 후보로 제안" 경로가 실제로 작동한 사례
- 컴파일과 시뮬레이션은 수행하지 않았다. 모든 finding은 코드 추적만으로 얻은 것이며, `observe` 항목이 그 확인 방법이다
