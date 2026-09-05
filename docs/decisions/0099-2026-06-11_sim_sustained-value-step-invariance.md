# 0099 - A Correction to Sustained Mode's value Semantics: the Window Total Divided by N_steps (step-size Invariance)

**Date**: 2026-06-11
**Status**: The N_steps formula has been superseded by [0131](0131-2026-07-13_model_sustained-value-per-day-not-per-span.md) (2026-07-13): "the total across the whole effective window" changed to "each matching day independently delivers the full amount," and how many days `date_range`/`days` covers no longer enters the `N_steps` computation, becoming only a hit filter. This ADR's other invariant (step_size only affects precision, never the result) is kept by 0131.
**Category**: Simulation engine / optimizer schema

---

## Background

The `mode: sustained` implemented by [0098](0098-2026-06-11_sim_optimizer-schedule-sustained-mode.md) currently writes `value` as-is at every hit step (`current + ev['value']`, with no scaling applied at all).

This creates a problem conflicting with a core framework principle: `step_size` should only be a precision parameter and should never change a model's physical result (this principle is implicit in the Euler discrete integration chapter and ADR 0092's "bare unit/event quantity" rule). But under the current implementation:

```
total contribution = value x N_steps, where N_steps = the effective window's duration / step_size
```

That is, in the very same YAML, changing `step_size` from `hour` to `30min` doubles the sustained variable's total contribution to the `state`, violating the design goal that "step only affects precision," and also inconsistent with pulse's semantics: pulse's `value` is "a one-time total," unrelated to `step_size` (as long as the event still falls within some step).

## Decision

Change `mode: sustained`'s `value` field semantics to "the total quantity across the whole effective window" (the same dimension as pulse's "one-time total"), with the engine writing `value / N_steps` at each hit step:

```
N_steps = the effective window's total duration / step_size
per_step_value = value / N_steps
```

No change is needed on the `state`-formula side, continuing to follow the existing rule that a `type: input` variable's contribution to a `state` formula is not multiplied by `step` (model.md lines 388-405). Verification:

```
total contribution = per_step_value x N_steps = (value / N_steps) x N_steps = value
```

Unrelated to `step_size`, and self-consistent under the same formula as pulse (`N_steps=1`, so `value/1=value`); pulse is simply the special case of sustained at `N_steps=1`, not a separate piece of logic.

### The N_steps computation rule

`window_duration` is determined by `date_range` times `time_range`:

| `date_range` | `time_range` | `window_duration` |
|---|---|---|
| Given `[d0,d1]` | Given `[t0,t1]` | `(d1-d0+1 day) x (t1-t0)` |
| Given `[d0,d1]` | Omitted | `(d1-d0+1 day) x 24h` |
| Omitted (the whole run) | Given `[t0,t1]` | `total simulation days x (t1-t0)` |
| Omitted (the whole run) | Omitted | `total simulation duration` |

When `date_range` is omitted, the total simulation duration is needed, which is not available within a single call to `apply_regimens`; therefore `N_steps` must be precomputed once before the loop starts (once per sustained event, cached as `ev['_n_steps']`, without modifying the original YAML dict), rather than recomputed at every step.

## Implementation Key Points

1. `regimen_runner.py` gains `precompute_sustained_divisors(regimens, step_size_sec, total_steps, sim_start_date)`: iterating every event with `mode == 'sustained'`, computing `n_steps` per the table above (rounded to the nearest integer, minimum 1), and writing it into the event copy's `_n_steps` field.
2. The sustained branch in `apply_regimens`: `current + float(ev.get('value', 0)) / ev.get('_n_steps', 1)`.
3. Caller changes (calling the precomputation once, outside the main loop):
   - `session_manager.py`: at the entry of both simulation loops (real-time/batch).
   - `optimizer_engine.py`: before the `_run_sim` call (`total_steps` is already available in that function).
4. `docs/model.md`'s "mode: sustained" subsection gains the value-semantics explanation, the conversion formula, and an example.

## Implementation Record

- `regimen_runner.py`: added `_time_range_day_seconds`, `_n_active_days`, `precompute_sustained_divisors`; the sustained branch in `apply_regimens` now writes `value / ev['_n_steps']` (a pulse event's `_n_steps` defaults to 1, so its behavior is unchanged).
- `optimizer_engine.py` (`_run_sim`), `session_manager.py` (`start_session`): each calls `precompute_sustained_divisors` once before the simulation loop starts.
- `docs/model.md`: the "mode: sustained" subsection gains a "value semantics: a window total, adaptively split by N_steps" explanation.

## Impact and Validation

- An internal scenario file: the `optimize.value` of 4 sustained schedule entries has been recomputed per `N_steps` (144/264/360/360): `[0,3]` to `[0,432]`, `[0,2]` to `[0,528]`, `[0,1]` to `[0,360]`, `[0.1,0.8]` to `[36,288]`. An `--opt` smoke test (pop=8, gen=2): `success=True`, 5 solutions, `x` falling within the new bounds, `f` (patients_saved / health) numerically plausible, and the constraint `health>=10` satisfied.
- `models/test/test_sustained_mode.yaml`: `work_rate`/`recovery_rate`'s `optimize.value`/`value` recomputed per `N_steps=60` (multiplied by 60), so the per-step `work_rate`/`recovery_rate` variable values (and hence the trajectory) exactly match before the change. An `--opt` smoke test (pop=10, gen=3): `success=True`, 10 solutions, the work/fatigue trade-off between `cumulative_output`/`fatigue` matching expectations.
- Does not affect any existing pulse-only model (`N_steps=1` gives `value/1=value`, numerically unchanged).

## Known Out-of-Scope Issue (Not Handled in This ADR)

- `simulation.schedules` (`_apply_schedules`, the forward `--sim`/`plans` path) does not support `mode: sustained`; that path is an independent implementation (`model_structure/simulation.py`), recognizing only `pulse`/`step`/`linear` interpolation, and reading neither `mode` nor `time_range` nor `_n_steps`. `mode: sustained` currently only takes effect on the `optimization.schedules` and GUI regimen paths (both going through `apply_regimens`). If a forward-sim-only scenario (not running `--opt`) needs a sustained input, a separate ADR is needed to also wire `_apply_schedules` into `precompute_sustained_divisors`/`apply_regimens`'s logic, or to reuse the same N_steps computation.
- The smoke test found a pre-existing bug unrelated to this ADR: calling `run_optimizer(progress_callback=None)` causes NSGA-II to raise a `TypeError` because `callback=None` is treated as a callable (near `_run_nsga2` in `optimizer_engine.py`, where `_cb = _ProgressCb() if progress_callback else None` is then passed to `pymoo_minimize(..., callback=_cb)`). The GUI/CLI paths always pass a callback in practice and are unaffected; this only triggers when a script calls `run_optimizer()` directly without a callback.

## Relationship to 0100

[0100](0100-2026-06-11_sim_unify-pulse-sustained-time-interval.md)'s proposed "unified interval representation" reuses this ADR's `N_steps` division logic (pulse as the special case `N_steps=1`), so this ADR must be implemented before 0100.
