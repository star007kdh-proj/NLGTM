---
name: fix-and-compile
description: For each accepted finding, create a branch, apply the minimal patch, run the project's compile command, compare observable counters when a simulation flow exists, and open a PR with the finding as evidence. Never merges.
---

# Fix and compile

## Input

- Triaged `report.md` (rows marked `accept`), the matching entries in `findings.json` and `verdicts.json`.
- `blocks.yaml`: `compile`, optional `sim`, `branch_prefix`, `commit_style`.

## Procedure, per accepted finding

1. Branch from the run revision: `<branch_prefix>/<finding-id-lowercase>`.
2. **Minimal patch.** Change the guard, encoding, width, or progress condition the finding names. Do not refactor around it. Do not fix neighbouring code the finding did not name; put it in the PR body as a note.
3. Re-run the verifier's trace mentally on the patched code with the failing input. The required result must now hold. Write this as the patch rationale.
4. Run `compile`. A compile failure is fixed or the finding is reported as `fix blocked` with the error text. Never leave a non-compiling branch.
5. If `sim` is defined and the finding has `observe`, run the named workload before and after, and put the counter values in the PR body as a two-row table. If the counter does not move as predicted, say so and mark the PR draft.
6. Commit following `commit_style`. Body: what broke, on which input, what the patch changes. Cite the finding id.
7. Open the PR. Body: triage row, failing input, patch rationale, counters, and the exact revision reviewed.

## Rules

- One finding, one branch, one PR. Findings that must be fixed together are stated in the report and get one PR with both ids.
- Never modify `requirements/*.md` or `conventions.md`.
- Never merge. Never force-push.
- If the fix requires a design decision (two valid semantics), stop, describe both in the PR body, and mark the PR draft.

## PR body template

```
Finding: <id>  (requirement <R-id>, severity <sev>)
Reviewed revision: <rev>

Failing input: ...
Wrong result: ...
Patch: ...

Compile: <command> OK
Observe: <counter> before <v0> / after <v1>   (or: no sim flow)
```
