# <BLOCK> 요구사항
- source: mas/<block>.md @ <rev>
- review: pending | stable (<date>)

이 파일은 Agent(extract-requirements)가 생성한다. 사용자는 `status:`만 바꾼다.

- accepted : 맞다. Agent는 이 줄을 다시 건드리지 않는다
- edited   : 문장을 고쳤다. Agent가 의미 변화를 반영하고 accepted로 바꾼다
- rejected : 틀렸다. `why:`에 한 줄. 의도된 동작이면 intentional 블록으로 옮겨져 반례 탐색에서 제외된다
- owner    : 사용자가 직접 추가한 줄. accepted와 같이 취급

- [R-<BLOCK>-1] <Y일 때 X가 반드시 일어나야 한다 / X는 절대 일어나면 안 된다>
  - evidence: <file:line> | "<주석, assert 메시지, 카운터 이름, 커밋 제목 인용>"
  - observe: <위반 시 움직일 perf counter>        (선택)
  - status: proposed
- [R-<BLOCK>-2] ...
  - evidence: ...
  - status: proposed

## intentional
- [R-<BLOCK>-k] <옮겨진 줄> — <사용자의 이유>
