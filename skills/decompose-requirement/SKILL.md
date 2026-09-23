---
name: decompose-requirement
description: Expand one short requirement into a list of concrete, independently checkable claims by crossing it with the project's decomposition axes, trigger paths, sibling consumers, boundaries, and liveness. Produces claims.json for the refute-claim phase.
---

# Decompose a requirement into claims

## Input

- One requirement line whose `status` is `accepted` or `owner`: `[R-<BLOCK>-<n>] <text>`, optional `observe:`. Lines under `## intentional` are skipped and listed as `skipped: intentional`.
- `blocks.yaml` entry for the block (paths, docs).
- `conventions.md` (decomposition axes, sibling path list).

## Procedure

1. **Locate the mechanism.** Grep the block paths for the nouns in the requirement. Identify the trigger (what event should cause the effect), the state that carries the decision, and the effect (the write, the flush, the output). Record `file:line` for each.
2. **Cross with axes.** For each decomposition axis in `conventions.md`, decide whether it changes the path from trigger to effect. If yes, one claim per class. If no, note `n/a` with a one-line reason.
3. **Sibling consumers.** List every path that reads the same state as the trigger path. For each, one claim of the form "sibling path applies the same guard / uses the same encoding as the trigger path".
4. **Boundaries.** One claim each for: field widths match between producer and consumer; index wrap; saturation; reset value; registers without reset that carry stale content.
5. **Liveness.** If the requirement implies "eventually" or "periodically", one claim: the effect completes in bounded time under continuous load (every cycle busy).
6. **Isolation.** One claim: the effect touches only the intended entry/way/slot.
7. Drop claims already covered by an `intentional:` note. Keep the id and mark `skipped: intentional`.

## Output: `claims.json`

```json
[
  {
    "id": "C-MBTB-1-3",
    "requirement": "R-MBTB-1",
    "text": "When the fetch block starts in the second align bank, the invalidate way mask selects the way that produced the prediction.",
    "axis": "fetch block start bank = 1",
    "trigger": "src/.../Ftq.scala:461",
    "effect": "src/.../MainBtb.scala:398",
    "kind": "state-class | sibling | boundary | liveness | isolation"
  }
]
```

## Quality bar

- Every claim names a concrete state class or path, never "in general".
- A requirement with fewer than four claims is under-decomposed unless the mechanism is a single assignment. Say why.
- Claims are independent. A refuter must be able to check one without reading the others.

## Anti-patterns

- Restating the requirement as a claim.
- Claims about code quality or naming.
- Guessing paths without `file:line`.
