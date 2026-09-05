# 0131 - A Correction to Sustained value Semantics: Each Matching Day Independently Delivers the Full Amount, Superseding 0099's "Total Across the Whole Span"

**Date**: 2026-07-13
**Status**: Implemented (supersedes 0099's N_steps formula, supersedes item 3 of 0126)
**Category**: Simulation engine / regimen semantics

---

## Background

ADR 0099 defined sustained's `value` as "the total quantity across the whole effective window": `N_steps = the effective window's total duration / step_size`, where "the effective window's total duration" equals the number of matching days `date_range` covers times the daily window width (`_n_active_days` times `_time_range_day_seconds`). ADR 0126 (2026-07-09) item 3 reconfirmed this rule. The goal of this design was that "`step_size` should only affect precision, never the total contribution"; under a fixed number of active days, this goal was indeed met.

While verifying the S1 Banister numeric-precision validation on 2026-07-13, the user pointed out that this definition has a counterintuitive side effect: `value`'s accumulated total is inversely proportional to how many days actually matched. For the same `value`, the more days `date_range` covers, the smaller the actual amount spread across each day; changing the simulation's total duration, `date_range`, or the `days` filter, without touching `value`, silently changes how much is actually delivered per day. This is the opposite of a common modeling intent, a "daily rate" (such as "a constant daily training load of 50 units, no matter how many days this plan runs"): a genuine daily rate should mean the per-day delivered amount stays fixed while the total scales proportionally with the number of days, not the total staying fixed while the per-day delivered amount scales inversely with the number of days.

Key evidence: the `sustained` plan in `models/test/valid/test_sustained_mode.yaml` had always carried a `label` field reading "same daily totals (work +60/day...) final cumulative_output/fatigue should match the pulse plan," but that plan never set a `date_range` (defaulting to cover the entire 5-day simulation), and under ADR 0099's formula, `value: 60.0` would be divided by "the total step count across the whole 5-day window," so it would actually deliver only `60/(5x12)=1/hr`, accumulating to just 60 over 5 days, not the "sustained over 5 days = 60" that the fixture itself expected and that ADR 0100's "Implementation Record" had already measured and honestly recorded at the time; this line of the record is itself direct evidence that this bug existed from day one of ADR 0099/0100's rollout, and simply no one had ever cross-checked it against the fixture's own stated expectation.

## Decision

Change sustained's `value` to "the window total independently delivered on each matching day," unrelated to how many days this entry actually matched across the entire `date_range`/simulation span:

```
N_steps = the single hit window's duration / step_size
        = _time_range_day_seconds(time_start, time_end) / step_size
per_step_value = value / N_steps
```

"Active days" is no longer computed (the `_n_active_days` function is removed entirely). `days` (the day-of-week filter) and `date_range`/`valid_start`/`valid_end` (the calendar interval) are downgraded to pure "does this hit" filters, only deciding whether a given day triggers, no longer taking part in the `N_steps` computation, exactly symmetric with a pulse event's `days`/`date_range` (a pulse has always meant "deliver the full `value` when it hits, 0 when it doesn't," never split across days).

No change is needed on the `state`-formula side, continuing to follow the existing rule that a `type: input` variable is not multiplied by `step`. Verification:

```
Each matching day's accumulated contribution = per_step_value x N_steps = value
```

Unrelated to `step_size` (ADR 0099's goal is preserved), and unrelated to how many days matched (a newly added invariant). The `sustained` plan in `test_sustained_mode.yaml`, using `value: 60.0` unchanged, now directly gives the correct result of 60/day, 300 over 5 days, under this new rule, matching the `pulse` plan exactly; see "Impact and Validation" below for the actual test.

## Relationship to 0099/0126

- Supersedes ADR 0099's N_steps formula ("the total across the whole span"), keeping its "step_size only affects precision" invariant, and adding a new invariant that "the number of matching days does not affect the per-day delivered amount."
- Supersedes ADR 0126 item 3 ("sustained's value represents the total quantity across the whole effective window... this requires the effective window's total duration to already be a determinate number at load time"), which describes exactly the old rule this ADR now changes; ADR 0126's other three items (the pulse-decay positioning, keeping the status quo for multi-entry override/accumulation, and handling `valid_range` date alignment independently) are unaffected.
- The originally planned direction of "adding a new `rate` field to regimen" (see the archived internal task `2026-07-13_task_step-size-adaptive-input-rate-design.md`) is no longer needed: after this ADR's fix, `value` itself now naturally carries the "daily rate" semantics, with no need for a separate parallel field.

## Implementation Record

- `reference_engine/src/schedule_runner.py`: removed `_n_active_days` (along with the `math` import); `precompute_sustained_divisors`'s signature simplified to `(schedules, step_size_sec)` (dropping the no-longer-needed `total_steps`/`sim_start_date`), with `_n_steps` now simply `day_sec / step_size_sec`; `apply_schedules`'s docstring rewritten to match, "each matching day independently delivers the full amount."
- The 4 call sites (`session_manager.py`, `reference_engine.py` x2, `optimizer_eval.py`) updated accordingly, dropping the now-unneeded `total_steps`/`sim_start_date` arguments at the call.
- `docs/model.md`: the value-semantics chapter, the N_steps formula table, and the window-width default rules section rewritten to the new rule, with a new multi-day repeated-trigger regimen example added (distinguishing the old "fixed total" semantics from the new "fixed daily rate" semantics, and stating clearly that the latter is now the only supported one).

## Migration: value/optimize.value in 5 Affected Files

Audit scope: entries in `simulation.plans[*].regimens` plus `optimization.startpoint.regimens` where `time_start != time_end` (sustained, not pulse) and, under the old rule, "matched days > 1."

Core judgment (checked entry by entry, not blindly divided by the old active-day count): does the entry's `label`/comment carry a trace such as "= daily rate x old N_steps," proving the author had manually multiplied the daily rate into a total under ADR 0099? If so, the current `value` is "a total the old rule forced to be hand-computed," and needs to be divided by the old active-day count to recover the daily rate; if not (the `value` was already the author's own daily rate, just silently diluted or concentrated by ADR 0099's bug), it is left untouched, and under the new rule this number is automatically correct.

| File | Handling | Note |
|---|---|---|
| `models/test/valid/test_sustained_mode.yaml` | The plan-level `sustained.work_rate/recovery_rate` unchanged (60.0/24.0); the two `optimization.startpoint` entries divided by the old active-day count of 5 (`[60,600]` becomes `[12,120]`, `120` becomes `24`) | The plan-level comment ("5/hour," "same daily totals") proves 60/24 was already the daily rate, and ADR 0099's bug had been diluting it all along, not something the author had pre-multiplied; the optimization section's comment explicitly states "old per-hour bounds x 60," proving it was pre-multiplied and needs dividing back |
| `models/papers/s3/burnout_allostatic/burnout_allostatic_sim.yaml` | All 21 `value` occurrences across 6 plans divided by their respective old active-day counts (86/62/24) | Every label carries a "x2064"/"x1488"/"x576" marker, directly proving these were totals hand-computed as "daily rate x old N_steps"; after dividing by the old active-day count, numeric validation (below) matched the old engine plus old values exactly |
| `models/papers/s3/sleep_schedule/sleep_schedule_sim.yaml` | All 22 `value` occurrences across 7 plans divided by their respective old active-day counts (28/20/8) | Same as above, labels carry "x672"/"x480"/"x192" |
| Another internal scenario file | 4 `optimize.value` occurrences in `optimization.startpoint.regimens` divided by their respective old active-day counts (6/11/15) | Comments such as "= [0,3] x 144" directly prove pre-multiplication |
| `models/test/valid/test_opt_t2.yaml` | Not changed, removed from the "affected files" list | The initial audit script misjudged these two T2 entries: the YAML writes no `time_start`/`time_end`, so statically it looks like "the all-day default," but both entries have an `optimize.time_start` interval search configured; when `optimizer_engine._build_regimen_events` decodes them, it recomputes `time_end` using the entry's own `_width_min` (1440 minutes, computed from the same default all-day window), which wraps all the way around back to `time_start`, so at run time this always degenerates into a pulse (`time_start==time_end`), never actually going through sustained's `N_steps` division, and was never affected by ADR 0099 |

Double-validation method: for the `burnout_allostatic_sim.yaml`/`sleep_schedule_sim.yaml` migration, `git stash` was used to switch back to "the old engine code plus old values" and run the baseline plan (`--sim`), then switch back to "the new engine plus new values" and run the same plan, comparing the final state field by field; `cortisol_chronic`/`cvd_risk`/`health_risk_index`/`cognitive_performance` and other values all matched exactly, confirming the migration did not change the numeric behavior of any already-published simulation result. `test_sustained_mode.yaml`'s `sustained` plan, run with the new engine, gives `cumulative_output=300.0` after 5 days, matching the `pulse` plan (it had been 60.0 before, exactly the number ADR 0100's original "Implementation Record" recorded, which this ADR has now judged to be the bug).

## Known Follow-On (Not Handled in This ADR, Left for a Separate Task)

- `test_plan.md`'s tier-1 numeric-precision validation protocol needs a new "step-size convergence testing" section added (a separate step alongside the comparison against the analytical solution).
- The three optimization scenarios `burnout_allostatic_opt_*`/`sleep_schedule_opt_*` need `--opt` rerun with the corrected startpoint bounds, checking whether the specific numbers already written into the S3 paper draft (`c_paper_s3_cn.md`; the workoutput/cvdrisk/joint front values, cognitive_performance/health_risk_index, etc.) change accordingly; these two scenarios' `optimize.value` semantics already depend on "searching one value per day, weekday and weekend independent," and this ADR's correction affects exactly the layer of "how a multi-day span gets split," which directly affects these two scenarios' T1 search-bound values.
- See the internal task `2026-07-13_task_s3-opt-rerun-after-adr0131.md` for detail (split out as its own still-active task from "batch 2 leftovers" in the archived internal task `2026-07-13_task_step-size-adaptive-input-rate-design.md`).
