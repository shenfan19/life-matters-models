# ADR 0097 - Simplifying papers/ Model description to Three Fields

## Status

Implemented

## Date

2026-06-08

## Background

ADR 0065 defined the structured description convention, recommending 9 fields: `brief`, `need`, `problem`, `method`, `simulation`, `optimization`, `result`, `conclusion`, `limitations`.

As the number of paper models grew, two problems surfaced in practice:

1. Too many fields: even at 2-3 sentences each, 9 fields still add up to 500-800 words, hard to skim quickly in the GUI.
2. Too academic a tone: `method` and `optimization` relied heavily on framework-internal notation (T1/T2/T3/T4, K×4, NSGA-II, Pareto), hard for a lay reader (a clinician, an interested non-specialist) to follow, while a researcher can already read the YAML's variables and formulas directly.

`description`'s primary audience is the GUI's non-specialist reader, not a substitute for the paper's own body text.

## Decision

Models under `papers/` uniformly adopt a three-field description structure:

```yaml
description:
  problem: >
    What it is, why it exists, and the core scientific tension, 3-5 sentences, understandable to a non-specialist reader.
    Framework-internal notation (T1/T2/T3/T4, K×4, NSGA-II, etc.) removed.
  result: >
    A summary of the simulation conclusion or Pareto front, 2-4 sentences;
    if not yet run, state the theoretical expectation and note "not yet run."
  limitations: >
    Known modeling boundaries and parameters still to be refined, 1-3 sentences.
```

Field-merging rule:
- `problem` = former `brief` + `need` (background motivation folded into the problem statement)
- `result` = former `result` + `conclusion` (the conclusion is an interpretation of the result, not a separate field)
- `limitations`: kept unchanged

Removed fields: `brief`, `need`, `method`, `simulation`, `optimization`, `conclusion`.

Scope: `papers/` (s1 through s5, and the top level). `references/` and other models are not constrained and may continue using any fields.

Top-level `clinical_brief`: merged into `description.problem` and no longer used as a top-level metadata field.

## Tone Requirements

- Remove framework-internal notation, describe things in plain medical/scientific language.
- Instead of "T1 intensity x T3 frequency x T4 timing," use "per-phase load intensity, weekly training sessions, escalation timing."
- Keep domain terminology (ALT, GFR, cortisol, Pareto front, etc.), which is already intuitive enough for the target reader.
- `result` may grow naturally with the Pareto front's complexity (anywhere from 2 to 5 sentences).

## Impact

- ADR 0065 remains in effect; this ADR narrows the recommended field set within `papers/`'s scope.
- All 15 YAML files under papers/ were migrated on 2026-06-08.
- `model.md`'s `metadata.description` specification paragraph has been updated to match.
- No GUI change needed: the field-order display logic is unchanged, and having fewer fields naturally makes it more compact.

## Non-Goals

- This does not force `references/` models to adopt the three-field structure.
- This does not turn the three fields into a hard schema constraint (the validator remains permissive).
- This does not restore method/optimization's technical detail inside description; that content belongs in a variable's or formula's `description` and `reference` fields.
