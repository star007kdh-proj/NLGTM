---
name: judge-and-report
description: Merge verified findings, assign severity, surface requirement candidates the requirements file does not cover, and write a designer-facing report.md with a triage table that can be filled in under five minutes.
---

# Judge and report

## Input

- `claims.json`, `findings.json`, `verdicts.json`.
- `requirements/<block>.md`, `blocks.yaml`.

## Procedure

1. Keep only findings with verdict `CONFIRMED`. Put `UNCERTAIN` in a separate section. Drop `REJECTED` but keep a one-line list with the rejection reason so the designer can see what was checked.
2. **Merge.** Two findings with the same mechanism and the same effect location are one finding. Keep the one with the more concrete input; list the other's input as an additional case.
3. **Severity.**
   - `functional`: required effect does not happen or hits the wrong target.
   - `liveness`: required effect happens only under conditions that continuous load can prevent.
   - `isolation`: unrelated state is modified.
   - `latent`: unreachable under the active config but reachable under a listed alternative config.
4. **Coverage.** For each requirement, list claims checked, findings, and NO_COUNTEREXAMPLE traces. A requirement with zero claims checked is reported as "not covered", never as "passes".
5. **Requirement candidates.** While reading the block, note mechanisms that no requirement mentions (a counter that is updated, a flush that is issued, a table that is trained). Phrase each as a one-line requirement the designer could paste in. Do not evaluate them.
6. Write `report.md`.

## `report.md` layout

```
# <block> review at <rev>

## Triage
| id | severity | requirement | one-line summary | file:line | accept / reject / intentional |
|---|---|---|---|---|---|

## Confirmed findings
### F-...  (severity)
what breaks / on which input / where / how to observe / suggested fix direction (one line)

## Uncertain
...

## Coverage
| requirement | claims | confirmed | no-counterexample | not covered |

## Checked and rejected
- F-... : <reason>

## Requirement candidates
- [R-<BLOCK>-?] <one line>
```

## Quality bar

- A designer can fill the triage column without opening the code.
- No finding appears in two sections.
- Coverage table rows sum to the claim count.
