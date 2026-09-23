# 실행 예시

각 디렉터리는 한 블록에 대한 전체 흐름(추출 → 리뷰 반복 → 감사 → 수정)의 산출물을 담는다. Agent가 생성한 파일은 그대로 두고, `NOTES_KR.md`에만 사람이 사후 설명을 붙인다.

```
examples/<name>/
  mas.md                   # extract-mas 출력
  requirements.v1.md       # extract-requirements 첫 출력 (전부 status: proposed)
  requirements.v2.md       # 사용자 리뷰 표시 후 iterate 출력. stable까지 버전 증가
  claims.json              # decompose-requirement 출력
  findings.json            # refute-claim 출력
  verdicts.json            # verify-finding 출력
  report.md                # judge-and-report 출력, triage 열 채움
  patches/*.diff           # fix-and-compile 출력
  NOTES_KR.md              # 정답 대조: 이후 실제 수정 커밋 또는 upstream PR과의 대응
```

목록
- `ittage-useful-reset/`: 실행 완료 (2026-09-23). finding 5건, 신규 2건. 정답 대조는 NOTES_KR.md
- `mbtb-pd-invalidate/`: 예정. 정답지는 BTB invalidate 수정 커밋 3건
- `backend-targetmem/`: 예정. 정답지는 targetMem 도입 커밋. ZeBu로 한 달 이상 걸린 건
- 이후 backend / mem 블록, 그리고 하드웨어가 아닌 프로젝트 예시 추가. 예시는 방법의 증거일 뿐 Agent는 블록이나 언어에 종속되지 않는다
