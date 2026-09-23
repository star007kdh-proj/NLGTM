---
name: extract-requirements
description: Derive short, purpose-level functional requirements for a block from its MAS and intent signals, write them with evidence and status:proposed into requirements/<block>.md, and in iterate mode reconcile the owner's accept/edit/reject/add marks until the file is stable. Owners review; they never have to write.
---

# Extract requirements

A requirement is a purpose stated as an obligation: "X must happen when Y" or "X must never happen". It is short, one or two sentences, never a paragraph. It says nothing about how. Detail is not the goal; a simplified requirement is enough to find bugs, because bugs live in the cases where it silently stops holding.

## Input

- `mas/<block>.md` from extract-mas.
- Existing `requirements/<block>.md` if present, with the owner's marks.
- `conventions.md` (for the vocabulary of state classes, so requirements use the owner's words).

## Procedure, first extraction

1. For each **mechanism** in the MAS, ask: what would be wrong if this mechanism did not fire, fired on the wrong entry, or fired too late? Write that as an obligation. One mechanism can yield zero, one, or two requirements. Skip mechanisms whose absence would only cost performance unless an intent signal says otherwise.
2. For each **intent signal** (comment, assert, counter, commit title), write the obligation it implies if not already covered. Quote the signal as evidence.
3. For each **state** with multiple writers, write an isolation obligation: a write for one entry must not affect another.
4. For each mechanism that runs "eventually" or "periodically", write a liveness obligation: it must complete in bounded time under continuous load.
5. Merge duplicates. Prefer the phrasing closest to the owner's vocabulary in `conventions.md`.
6. Number them `[R-<BLOCK>-<n>]`. Attach `evidence:` (one or two `file:line` or quoted signals), optional `observe:` (a counter from `blocks.yaml` that would move on violation), and `status: proposed`.
7. Write the file. Stop. The owner reviews.

## Procedure, iterate mode

1. Read the owner's marks. Status values the owner may set: `accepted`, `edited` (they changed the text), `rejected` (with optional `why:`), and new lines they added with `status: owner`.
2. `accepted`: freeze. Never touch the line again.
3. `edited`: re-read the new text. If it changed the meaning, check the MAS for mechanisms that now fall outside any requirement and propose lines for them. Set the edited line to `accepted`.
4. `rejected` with `why:`: if the reason says the behaviour is intentional, move the line to the `intentional:` block at the bottom so refuters skip it. If the reason says the requirement was wrong, drop it and look for the mechanism's real purpose in the MAS open questions; propose a replacement if one exists.
5. `owner`: treat as accepted. Check the MAS for evidence and attach `evidence:` if found; leave a note if not.
6. Re-run the first-extraction procedure only over MAS mechanisms not covered by any current line. Add proposals.
7. Stop when no line is `proposed` and step 6 added nothing. Report "stable".

## Output: `requirements/<block>.md`

```
# <BLOCK> 요구사항
- source: mas/<block>.md @ <rev>
- review: pending | stable (<date>)

- [R-<BLOCK>-1] <obligation, one or two sentences>
  - evidence: <file:line> | "<quoted signal>"
  - observe: <counter>                (optional)
  - status: proposed | accepted | edited | rejected | owner
  - why: <owner's reason>             (optional, on rejected)

## intentional
- [R-<BLOCK>-k] <line moved here> — <owner's reason>
```

## Quality bar

- Between three and ten lines for a typical block. More means the block is too big or the lines are mechanisms, not purposes.
- Every line is falsifiable: a refuter can imagine an input that would violate it.
- No line contains a signal name, a stage name, or a data structure name unless the owner uses that name in `conventions.md`.

## Anti-patterns

- "The block implements X." That is a description, not an obligation.
- Requirements about code quality, naming, or timing.
- Rewriting an accepted line, even to fix a typo. Propose a new line instead.
