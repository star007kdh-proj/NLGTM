---
name: verify-finding
description: Independently re-derive one alleged counterexample from the claim, the failing input, and the location only, without the refuter's reasoning. Returns CONFIRMED, REJECTED, or UNCERTAIN with a self-contained justification.
---

# Verify a finding

You are given a claim, a failing input, and a location. You are not given the argument. Rebuild it yourself. If you cannot, the finding is rejected.

## Input

- Claim text, `failing_input`, `location`, `required_result`. Nothing else from the finding.
- Project config paths, run revision.

## Procedure

1. Starting from `location`, trace the path with the given input yourself. Compute every guard and every comparison for that input.
2. **Reachability.** Can this input actually occur? Check upstream producers: is the state class real under the active configuration, or does a parameter constant-fold it away?
3. **Other guards.** Is the required effect produced by a different path the refuter may have missed (a sibling that already handles it, a later stage that repairs it, a flush that makes it harmless)?
4. **Severity of the wrong result.** Does the wrong result violate the requirement, or only differ from the refuter's expectation while the requirement is still met?
5. **Observation.** Would the named counter or signal actually show the divergence? If not, say what would.

## Output: one entry in `verdicts.json`

```json
{
  "finding": "F-MBTB-1-3-a",
  "verdict": "CONFIRMED | REJECTED | UNCERTAIN",
  "trace": "<your own derivation: guard values and comparisons for the given input, each with file:line>",
  "reason": "<one paragraph: why the requirement is or is not violated on this input>",
  "reachability": "reachable under <config> because <producer file:line>",
  "observe_ok": true
}
```

## Verdict rules

- `CONFIRMED`: you reproduced the wrong result independently and no other path supplies the required effect.
- `REJECTED`: the input is unreachable, or another path supplies the effect, or the result does not violate the requirement. State which.
- `UNCERTAIN`: you could not finish the trace. Say exactly where the trace stopped. Uncertain findings go to the report in a separate section, never as confirmed.

## Anti-patterns

- Confirming because the story is plausible. Compute the values.
- Rejecting because the fix would be hard.
- Reading the refuter's reasoning if it leaks into your input. Ignore it.
