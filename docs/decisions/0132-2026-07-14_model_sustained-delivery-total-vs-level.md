# 0132 - Adding delivery: total | level to a Sustained Regimen, Distinguishing "Splitting a Total" from "a Constant Level"

**Date**: 2026-07-14
**Status**: Implemented
**Category**: Simulation engine / regimen schema

---

## Background

ADR 0098's (2026-06-11, sustained mode's original proposal) original motivation was to express a decision variable such as sustained care intensity, sustained protection level, or a sustained rest fraction, one that should stay constant across several consecutive steps, literally a "level" semantics.

But later the same day, ADR 0099 redefined sustained's `value` as "the total quantity across the whole effective window, divided by `N_steps` and spread across each hit step," to solve a different, real problem (a change in `step_size` should not change the total contribution). This redefinition never went back to check whether it still satisfied ADR 0098's own example: after dividing by `N_steps`, a level that "should stay constant" instead has its reading change inversely with `step_size` (and, from ADR 0131 onward, with the number of matching days too). While a model had only ever been run at one fixed `step_size`, this contradiction was numerically invisible.

While validating ADR 0131 on 2026-07-14, testing more models with a variable-step-size grid found that `burnout_allostatic_sim.yaml`/`sleep_schedule_sim.yaml` diverged at a non-native `step_size` (see the internal reference [[project_sustained_level_vs_dose_gap]], archived as the internal task `2026-07-14_issue_sustained-input-level-vs-dose-semantics-gap.md`). Root-cause analysis confirmed: `sleep_hours`/`bedtime_hour`/`nutrition_score` (and the three decision variables in ADR 0098's own original validation case) are all read directly as an "instantaneous level" in their respective formulas (a multiplicative factor, or a deviation-from-baseline calculation), not used as an "accumulated total." This is exactly the semantics ADR 0098 originally intended to express, silently overridden by ADR 0099's "total divided by N_steps" rule; the two decisions had been contradicting each other since 2026-06-11 without anyone noticing.

Banister's `training_load` (a genuine daily rate, "50 AU/day, with the total scaling proportionally with the number of days") does genuinely need ADR 0099/0131's "split the total" semantics; not every sustained input should be a level, and both semantics are genuinely needed, the problem was only that there used to be just one option to choose from.

## Decision

A new optional field, `delivery: total | level`, is added to a regimen entry, defaulting to `total` (the current behavior, fully backward-compatible, requiring no migration of any existing model unaffected by this issue):

```yaml
regimens:
  - variable: sleep_hours
    time_start: "00:00"
    time_end: "24:00"
    value: 6.0              # directly the target level, no need to manually multiply by N_steps
    delivery: level          # each hit step delivers value directly, without dividing by N_steps
    days: [Mon, Tue, Wed, Thu, Fri]
```

| `delivery` | Semantics | Delivered amount per hit step | Applicable scenario |
|---|---|---|---|
| `total` (default) | The window's/matching day's total, split per ADR 0131 | `value / N_steps` | Training load, total food intake, total dose delivered, quantities that are inherently "how much was put in during this period," accumulating over time |
| `level` | A constant level, not split | `value` itself | Sleep duration, bedtime timing, a diet-quality score, care intensity, protection level, quantities that are inherently "the current state or setting," not accumulating over time |

`delivery: level` is a no-op for a single-step window (a pulse, `time_start==time_end`); a pulse is already the special case `N_steps=1`, so dividing or not dividing gives the same result, and no conflict arises; the two `delivery` values collapse into the same thing on a pulse.

## Relationship to Existing Principles

- **This is not reintroducing the pulse/sustained switch ADR 0100 removed**: what ADR 0100 removed was a redundant distinction ("point versus window" is fully determined by a single window-width number, so removing it lost no information). `total` versus `level` is a new, independent dimension: Banister's `training_load` and this ADR's fix for `sleep_hours`/`care_intensity` use exactly the same window width (all day), so window width alone cannot distinguish the two, and an explicit declaration is required.
- **This does not rely on a variable-naming convention** (such as an internal `_step` suffix): the path of "letting the engine special-case based on a variable name" was already explicitly rejected on 2026-07-09 (the existing principle that variable names are free and the engine recognizes no reserved names). This field is a declaration written explicitly on the regimen entry, at the same level as `time_start`/`time_end`/`days`/`date_range`, not a naming convention.

## Implementation

- `reference_engine/src/schedule_runner.py`: inside `apply_schedules`, `delta = value/n_steps` now branches on `ev.get('delivery', 'total')`, with `delta = value` delivered directly under `level`. `precompute_sustained_divisors` is unchanged (`_n_steps` is still precomputed; the `level` branch simply does not use it).
- `model_structure/loader.py`: `_parse_schedule_entries` passes the `delivery` field through (omitted by default).
- `optimizer_engine.py`: `_build_regimen_events` passes `delivery` through (under `level`, T1's `optimize.value` search bounds are directly the target level's lower and upper bounds, with no need to multiply by `N_steps`).

## Migration

The following models' `value`/`optimize.value` were restored from "a total manually multiplied by N_steps" back to a direct level value, with `delivery: level` added:

- `models/papers/s3/burnout_allostatic/burnout_allostatic_sim.yaml` (`sleep_hours`/`bedtime_hour`/`nutrition_score`, 21 occurrences)
- `models/papers/s3/sleep_schedule/sleep_schedule_sim.yaml` (the same three variables, 22 occurrences)
- Another internal scenario file (the same three variables, 4 occurrences of `optimize.value`)
- `burnout_allostatic_opt_{workoutput,cvdrisk,joint}.yaml`, `sleep_schedule_opt_{cognitive,healthrisk,joint}.yaml` (each with its own `optimization.startpoint.regimens` bounds, the same batch of variables)

`exercise_min`/`break_min`/`nap_minutes` (pulses) and `training_load` (Banister, a genuine daily rate) are unaffected and keep `delivery: total` (the default, no declaration needed).

Validation: with a variable-step-size grid (1h/30min/15min), `sleep_hours`'s reading stayed constant and no longer drifted with step size; `burnout_allostatic_sim.yaml`/`sleep_schedule_sim.yaml`'s `--sim` results matched exactly with the pre-migration numbers (at the native `step_size=1h`), and no longer diverged at a non-native step size. See the archived internal task `2026-07-14_issue_sustained-input-level-vs-dose-semantics-gap.md` for the full validation record.

## Known Limitation (Not Handled in This ADR)

- Whether other models under `models/` still have this same pattern of "a sustained input read as a level" has not been systematically checked; only the known trigger cases (burnout_allostatic/sleep_schedule/another internal scenario file) were investigated, without a full AST scan across all of `models/`, left as a follow-up task.
