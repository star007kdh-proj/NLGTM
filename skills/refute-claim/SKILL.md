---
name: refute-claim
description: Given one checkable claim, trace the code from trigger to effect and construct a concrete input on which the claim fails. Emits a finding with location, failing input, wrong result, and observation method, or NO_COUNTEREXAMPLE with the trace that shows the claim holds.
---

# Refute a claim

Your job is to break the claim. Assume it is false and look for the input that proves it.

## Input

- One claim (JSON from `claims.json`).
- Project config paths, run revision.

## Procedure

1. Read the trigger and effect locations. Read outward until you hold the full path: every `when`, every `Mux`, every valid/ready, every register between them.
2. Write down the **guards**: each condition that must hold for the effect to fire. For each guard, ask: is there an input in this claim's state class where the guard is false although the requirement says the effect must happen?
3. Write down the **encodings**: each comparison between a stored value and a computed value. Confirm both sides are built by the same convention (`conventions.md`). Logical vs physical index, aligned vs full offset, tag width, pointer flag bits.
4. Check **width and range**: masks vs field widths, padding bits, counter wrap, saturation at the wrong edge.
5. Check **stale state**: registers without reset or without a clear on the path. Can a previous occupant's value be consumed?
6. Check **liveness** when the claim is a liveness claim: what makes progress, and can that condition be false every cycle under continuous load?
7. Check **isolation**: does the effect address exactly one entry/way/slot, or can it hit neighbours (per-way vs per-set, mask vs index)?
8. If you found a break, construct the smallest concrete input: which state class, which values, what the guard/encoding evaluates to, what happens instead of the required effect.
9. Name how a designer would observe it: a perf counter that would diverge, or a signal (`file:line`) to watch in a waveform and what value it takes.

## Output: one entry in `findings.json`

```json
{
  "id": "F-MBTB-1-3-a",
  "claim": "C-MBTB-1-3",
  "requirement": "R-MBTB-1",
  "verdict": "COUNTEREXAMPLE",
  "location": ["<rev>:src/.../MainBtb.scala:398-401"],
  "failing_input": "fetch block starts at VA[5]=1; prediction came from way 2 of bank 1 with stored position high bit = 0 (logical order)",
  "wrong_result": "compare value uses physical bank bit 1; all four ways mismatch; wayMask = 0; entry survives",
  "required_result": "wayMask selects way 2",
  "observe": "counter pd_invalidate stays flat while ifuRedirect with attribute None repeats on the same block; or watch wayMask at MainBtb.scala:402",
  "mechanism": "encoding | guard | width | stale | liveness | isolation"
}
```

Or:

```json
{ "claim": "C-...", "verdict": "NO_COUNTEREXAMPLE", "trace": "<guards and encodings checked, each with file:line, and why each holds>" }
```

## Quality bar

- `failing_input` is concrete enough that another agent can re-derive `wrong_result` without your reasoning.
- Every guard and encoding you relied on has a `file:line`.
- One finding per distinct mechanism. Do not pad.

## Anti-patterns

- "This looks suspicious." Either construct the input or return NO_COUNTEREXAMPLE.
- Reporting behaviour that is gated off under the active configuration.
- Estimating performance impact.
