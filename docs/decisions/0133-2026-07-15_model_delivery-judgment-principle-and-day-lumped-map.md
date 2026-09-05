# 0133 - The delivery: total/level Decision Principle, Plus Identifying the "day-lumped map" Anti-Pattern

**Date**: 2026-07-15
**Status**: The decision principle is settled; the anti-pattern fix (the sleep_hours/bedtime_hour refactor) is not implemented, see the task link below
**Category**: Modeling methodology / regimen schema (extending [0132](0132-2026-07-14_model_sustained-delivery-total-vs-level.md))

---

## Background

ADR 0132 introduced `delivery: total | level`, fixing a numeric symptom, that a reading inside a sustained window should not be diluted by `N_steps`. But it did not answer how a modeler should decide which `delivery` a given `type: input` variable needs; the existing description ("accumulated downstream versus read directly as a coefficient") is not precise enough and can mask a deeper problem: some variables appear on the surface to "need level," but the real reason is not that variable's own physical nature, it is that the downstream formula itself was written at a coarse granularity (`step_unit: day`) as a one-shot map rather than a step-by-step integrable dynamics equation. In that case, `delivery: level` merely redistributes "the answer computed once for the day" to every finer step; this is not, by itself, a wrong reading, but it is also not genuine step-by-step integration.

A `life-matters-reference-engine` session on 2026-07-15 checked every `delivery: level` use case across the whole repository one by one (`running_2026.yaml`'s `training_intensity`/`pace`; three decision variables in an earlier internal scenario file; `sleep_schedule_sim.yaml`/`burnout_allostatic_sim.yaml`'s `sleep_hours`/`bedtime_hour`/`nutrition_score`), and found these cases split clearly into two groups: only one group is a variable that physically has no reasonable reading other than level, and the other group is a formula written at the wrong granularity, patched over with level.

## Decision

### 1. Terminology clarification (to avoid later discussion conflating three independent axes)

| Concept | Layer | What it governs |
|---|---|---|
| `sustained` | The regimen input mechanism (ADR 0127) | The sole delivery mechanism for `type: input`; a pulse (`time_start==time_end`, `N_steps=1`) is its special case, not a second, parallel mode |
| `delivery: total \| level` | A regimen-entry field (ADR 0132) | When a sustained window hits, whether each step delivers `value/N_steps` (total) or `value` itself (level) |
| `step_unit` | A field on a formula's `dynamics` block | How much time this formula's own `step` variable corresponds to, entirely unrelated to regimen/delivery, a formula-layer concept, not an input-layer one |

The three belong to different layers and may all appear on the same variable at once, but none is an alias or subset of another.

### 2. The `delivery` decision rule (superseding ADR 0132's original "accumulate versus read directly" description)

Rule A: does this variable participate in a downstream formula as a coefficient or an instantaneous state, in a formula written at native step granularity (with a `step_unit` no coarser than `simulation.step_size`)?
- If yes: use `delivery: level`, the sole correct, final answer requiring no further treatment. Examples: `training_intensity`/`pace` (read natively minute by minute in `heart_rate_response`/`fatigue_accumulate`), `care_intensity`/`self_protection`/`rest_hours` (read natively hour by hour in `radiation_accumulation`/`rest_slows_ars`). Such variables physically have no "total" dimension at all (asking "what is training_intensity's total" is meaningless), so there is no ambiguity.
- Is the variable itself "how much was put in this time," accumulated by a downstream state? Then use `delivery: total` (the default).

Rule B (new), the "day-lumped map" anti-pattern test: if a variable, in order to display a "level" effect, forces its downstream formula to use a `step_unit` coarser than `simulation.step_size` (such as `day`) to compute the net change across several simulation steps in one shot, then relies on `delivery: level` to redistribute this "one-shot answer" unchanged to every finer simulation step, this is a design-smell signal that cannot be fixed by adjusting `delivery`; it means the variable was modeled with the wrong primitive to begin with. The correct direction is to split it into an instantaneous indicator readable at native granularity (a true level, such as `is_asleep`) plus a derived state accumulated from it (a true total, such as `sleep_hours_today`, following the same pattern as `lm_score`), with the formula rewritten accordingly as a genuine ODE at its native `step_unit`.

Current hits: `sleep_hours`/`bedtime_hour` (in `sleep_schedule_sim.yaml`/`burnout_allostatic_sim.yaml`, where `sleep_pressure_dynamics` declares `step_unit: day` yet uses an all-day sustained input plus `delivery: level`, repeatedly reading the same daily quantity hour by hour). `nutrition_score` is suspected of the same pattern, not individually verified.

## Relationship to Existing Principles

- Extends rather than overturns ADR 0132: 0132's field definition, engine implementation (`schedule_runner.py`), and existing migration record are all kept unchanged; this ADR only adds a layer of methodology on how to decide which to use and when to suspect `delivery` is treating a symptom rather than the cause.
- Echoes the lesson from the Banister `*step` review (the investigation record that preceded ADR 0131): "tested under only one granularity, so the problem never surfaced." Rule B essentially distills that lesson into an executable check, to prevent the same anti-pattern from recurring unnoticed in other models.
- Impact on `draft_s1_numerical_consistency.md` (an S1 paper candidate subsection): the "numeric consistency guarantee" argued there holds only for rule-A-type models (formulas written at native granularity); for rule-B-type models (a day-lumped map), the reading does not drift with `step_size` (already validated by ADR 0132), but the equation itself does not converge to a more precise solution as the step size is refined. These are two different strengths of "consistency," and the paper needs to distinguish them; this has been recorded as a task (see below).

## Current State and Cost (the Unimplemented Part)

- Rules A/B, as decision principles, are settled by this ADR; writing them into the "delivery decision rule" section of `docs/model.md` is pending execution.
- The `sleep_hours`/`bedtime_hour` anti-pattern rule B identified is outside this ADR's fix scope; the fix (an `is_asleep` state plus a pulse pair, in place of a duration-type input) has been recorded as a separate task, involving rewriting the formula and recalibrating the S3 paper's numbers, with whether and when to execute it left to that task's own decision; see the internal task `2026-07-15_task_sleep-model-native-step-reform.md` for detail.
- The current numeric fix `delivery: level` provides for `sleep_hours` (ADR 0132) is unaffected and remains valid, already validated as step-size-robust (no divergence across a 1h/30min/15min regression), just not solving the finer-grained dynamics-expression problem.

## Implementation

- `docs/authoring/regimens_and_optimization.md` (the successor path to `docs/model.md`): the "delivery decision rule: a structural-position test, also catching the day-lumped-map anti-pattern (ADR 0133)" section has been added, the full version of rules A plus B, replacing the previously vaguer "accumulate versus read directly" description; the same document adds a "conceptual foundation" section unifying `value`/`delivery`/`days`/`date_range` under the extensive/intensive-quantity framework (2026-08-25, landed together with the same batch of rewrites for S1 paper section 4.4; see that section's equivalent argument).
- `draft_s1_numerical_consistency.md` / S1 paper section 4.4: the argument now covers only rule-A-type models (formulas written at native granularity); for rule-B-type models (a day-lumped map), the reading does not drift with `step_size`, but the equation itself does not converge to a more precise solution, and this rewrite (2026-08-25) keeps only one premise-stating sentence in the paper without expanding into the anti-pattern's full technical description (after review, the user judged the paper's body text should understate the example, keeping the full anti-pattern description in this ADR and the authoring document above). The `sleep_schedule`/`burnout_allostatic` papers' `limitations` paragraphs have not yet been given a more precise description, left for when those two papers are next touched.

## Known Limitation (Not Handled in This ADR)

- A repository-wide AST scan for "are there other models hitting the day-lumped-map anti-pattern" has not been done; only the currently known `delivery: level` hits (10 files) have been checked manually.
- Whether `nutrition_score` is a rule-B hit (versus rule A) has not been individually verified against its downstream formula, left to be confirmed the next time `burnout_allostatic_sim.yaml` is touched.
