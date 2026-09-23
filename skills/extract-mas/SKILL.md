---
name: extract-mas
description: Read one code block and write a minimal architecture summary (MAS): what the block is for, its interfaces, the state it owns, and the mechanisms that change that state, each with evidence locations. The MAS is the input to requirement extraction, not a specification.
---

# Extract a minimal architecture summary

The MAS is short on purpose. It exists so that requirements can be derived and so the owner can check them quickly. It is not documentation.

## Input

- `blocks.yaml` entry: paths, docs, observe counters.
- `conventions.md`.
- Anything in the paths, plus: docs listed, comments, assert/error messages, perf counter names, test names, and `git log --oneline -- <paths>` (last 50).

## Procedure

1. **Purpose.** One or two sentences: what this block is for, from the point of view of its consumers. Draw from docs, file headers, and the names of its outputs. Cite the evidence.
2. **Interfaces.** Each input and output port group, one line each: who sends it, what it carries, what the block does on it. `file:line` of the definition.
3. **State.** Each storage the block owns (tables, queues, registers with history, pointers): one line each with size, index scheme, and which mechanisms read and write it.
4. **Mechanisms.** Each thing that changes state or produces an output: one line each in the form `trigger → effect`, with `file:line` for both ends. Include mechanisms that are gated by configuration and say which parameter gates them.
5. **Intent signals.** List every comment, assert (`XSError`, `assert`), perf counter, and commit message that states what the block *should* do. Quote it, cite it. These are the raw material for requirements.
6. **Open questions.** Anything the code does whose purpose is not stated anywhere. One line each.

## Output: `mas/<block>.md` (also copied to `out/<block>/<rev>/mas.md`)

```
# <block> MAS  (rev <rev>)

## Purpose
...

## Interfaces
- <port group>: from <who>, carries <what>; block does <what>   [file:line]

## State
- <name>: <size/index>; read by <mechanisms>; written by <mechanisms>   [file:line]

## Mechanisms
- M1 <trigger> → <effect>   [trigger file:line → effect file:line]   (gated by <param>?)

## Intent signals
- "<quoted comment / assert message / counter name / commit title>"   [file:line or commit]

## Open questions
- <what the code does with no stated purpose>
```

## Quality bar

- Under 120 lines for a typical block. If longer, the block should be split in `blocks.yaml`.
- Every line has a location. No line describes how something is implemented beyond trigger → effect.
- Mechanisms are complete: every write to a state entry appears in some mechanism.

## Anti-patterns

- Copying the code structure file by file.
- Describing algorithms in prose. The requirement phase needs purposes, not procedures.
- Omitting mechanisms because they look trivial. Trivial mechanisms are where asymmetries hide.
