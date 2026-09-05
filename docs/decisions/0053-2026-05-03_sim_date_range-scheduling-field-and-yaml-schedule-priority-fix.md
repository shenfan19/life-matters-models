# ADR 0053 - The `date_range` Scheduling Field and a Fix to YAML Schedule Priority

**Date**: 2026-05-03
**Status**: Implemented

---

## Background

ADR 0052 established `simulation.schedules`'s flat-list format (an HH:MM time plus a `days` day-of-week mask), but three problems remained:

### Problem 1: a false loop from listing entries week by week

To express "a different training load each week," a modeler would write out all 18 weeks' entries in the YAML:

```yaml
schedules:
  - variable: training_load
    value: 50.0
    days: [Mon,Tue,Wed,Thu,Fri]
    date_range: "2026-01-01 ~ 2026-01-07"   # week 1
  - variable: training_load
    value: 55.0
    days: [Mon,Tue,Wed,Thu,Fri]
    date_range: "2026-01-08 ~ 2026-01-14"   # week 2
  # ... repeated 16 more times
```

This produces a large amount of redundancy, and the engine places no limit on the entry count.

What was actually needed was "one entry effective only within a specified date range," that is, the `date_range` field. This field already existed in ADR 0052's YAML, but neither the frontend nor the backend fully implemented it.

### Problem 2: the frontend does not parse date_range

When `Simulator.tsx` reads a schedule entry, it only recognizes `s.valid_start` / `s.valid_end`, not `date_range`:

```typescript
validRangeEnabled: !!(s.valid_start || s.valid_end),  // date_range is ignored
```

Result: the date-range column in the GUI always displays empty, and the user cannot see or edit a validity period.

### Problem 3: a YAML Schedule is overridden by a GUI Regimen value

On every batch step, the frontend sends the current `inputParams` (every input variable's GUI default value) to the backend together with `input_changes`:

```python
# batch_steps, the old code
if input_changes:
    for var_name, value in input_changes.items():
        model.set_variable_value(var_name, value)
        model.manual_overrides[var_name] = value   # <- written into manual_overrides
```

`model.manual_overrides` makes `_apply_schedules()` skip that variable's YAML Schedule. Result:

- `test_ckd_protein`: the three-meal pulse (0.27 + 0.27 + 0.26 = 0.80 g/kg/day) gets overridden to the last event's value, 0.26.
- `test_glucose_meal`: `carb_intake` should return to zero between meals (pulse mode), but instead the previous meal's value stayed and kept accumulating, sending blood glucose straight to its upper bound.

### Problem 4: an epoch error in `_apply_regimens`

When `valid_range_enabled=true`, the backend used the fixed epoch 1900-01-01 to compute the "current simulation date":

```python
_EPOCH = date(1900, 1, 1)
sim_date = _EPOCH + timedelta(days=prev_day_idx)  # reaches at most 1900 plus a few decades
```

But `valid_start` is "2026-01-01," so `sim_date < date(2026, 1, 1)` is always true, and every event with a validity restriction is skipped entirely.

---

## Decision

### Decision 1: date_range as the standard date-range field for a schedule entry

Format: `date_range: "YYYY-MM-DD ~ YYYY-MM-DD"` (consistent with the global date-input convention; see ADR 0006 / global_prompt)

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2026-05-06"
  schedules:
    - variable: training_load
      time: "09:00"
      value: 70.0
      days: [Mon, Tue, Wed, Thu, Fri]
      date_range: "2026-01-01 ~ 2026-01-28"   # in effect only during weeks 1-4
      label: "Base-phase training"
    - variable: training_load
      time: "09:00"
      value: 100.0
      days: [Mon, Tue, Wed, Thu, Fri]
      date_range: "2026-01-29 ~ 2026-02-25"   # in effect only during weeks 5-8
      label: "Build-phase training"
```

The Python engine (loader.py) already supports `date_range` (implemented during ADR 0052); this ADR completes its implementation on the frontend and the regimen side.

When `date_range` is absent: the event is in effect every day for the entire simulation period (`start_date` to `end_date`).

### Decision 2: the frontend parses date_range into validStart / validEnd

`Simulator.tsx`'s schedule-entry parsing gains a fallback:

```typescript
let validStart = s.valid_start ?? '';
let validEnd   = s.valid_end   ?? '';
if (!validStart && !validEnd && s.date_range) {
  const parts = String(s.date_range).split('~');
  if (parts.length === 2) {
    validStart = parts[0].trim();
    validEnd   = parts[1].trim();
  }
}
```

`valid_start` / `valid_end` are still kept as forward-compatible equivalent aliases.

### Decision 3: a YAML Schedule takes priority over a GUI Regimen

Core principle: `simulation.schedules` is the model's behavior definition; a GUI Regimen is the user's interactive override. When the two conflict, the former takes priority.

Fix: `batch_steps`'s `input_changes` handling no longer writes to `manual_overrides`:

```python
# After the fix: only updates the initial value, without marking it as a manual override
if input_changes:
    for var_name, value in input_changes.items():
        if var_name in model.variables:
            model.set_variable_value(var_name, value)
            # manual_overrides is not written -> _apply_schedules runs normally
```

`manual_overrides` now has only two write paths:
1. The optimizer's `_run_sim()`: explicitly suppresses the YAML Schedule, letting the optimizer control that variable.
2. In the future: the user clicking "lock override" in the GUI (not yet implemented).

### Decision 4: `_apply_regimens` uses the model's actual start_date as its epoch

```python
# Fix: reads sim_start_date from the session
def _apply_regimens(model, regimens, prev_time, next_time, sim_start_date=''):
    try:
        _EPOCH = date.fromisoformat(sim_start_date) if sim_start_date else date(1900, 1, 1)
    except ValueError:
        _EPOCH = date(1900, 1, 1)
    # sim_date = _EPOCH + timedelta(days=prev_day_idx) -> now correctly matches the absolute calendar date
```

`sim_start_date` is read by `start_session()` from `base_model.simulator['start_date']` and stored into the session, then passed in by `batch_steps()`.

### Decision 5: test scenarios use the shortest cycle to validate the core logic

`test_banister.yaml` was simplified from 18 weeks (126 steps) to 1 week (6 steps), matching the other 5 test scenarios (`test_ckd_protein`, `test_glucose_meal`, L1/L2/L3); a test file only needs to verify the pipeline is correct, not reproduce a complete clinical protocol.

```yaml
# Before simplification: 18 date_range entries, 126 days
# After simplification: 2 entries, 7 days
simulation:
  start_date: "2026-01-01"
  end_date:   "2026-01-07"
  schedules:
    - variable: training_load
      time: "09:00"
      value: 70.0
      days: [Mon, Tue, Wed, Thu, Fri]
      date_range: "2026-01-01 ~ 2026-01-07"
    - variable: training_load
      time: "09:00"
      value: 35.0
      days: [Sat]
      date_range: "2026-01-01 ~ 2026-01-07"
```

---

## Priority Rule Summary

| Source | Write path | Priority | Purpose |
|------|---------|--------|------|
| The YAML `simulation.schedules` | `loader.py` to `model.schedules` to `_apply_schedules()` | Highest | The model's behavior definition |
| A GUI Regimen (`inputEvents`) | `batch_steps._apply_regimens()` | Medium (overridden by the layer above) | A user's interactive preview |
| An optimizer Regimen | `_run_sim()` plus `manual_overrides` | Highest (explicitly suppresses the schedule) | The optimization search space |
| `input_changes` (GUI default values) | `batch_steps`, only `set_variable_value` | Initialization only | Does not affect the schedule |

---

## Files Changed

```
sim_engine/src/simulator_engine.py
  _apply_regimens(): a new sim_start_date parameter; the epoch changed from 1900-01-01 to the model's start_date
  start_session(): the session now stores sim_start_date
  batch_steps(): input_changes no longer writes manual_overrides; passes sim_start_date to _apply_regimens

sim_gui/src/components/Simulator.tsx
  schedule-entry parsing: a date_range to validStart/validEnd fallback

models/scenarios/test/test_banister.yaml
  simplified to 1 week (7 days, 2 schedule entries)

docs/model_design.md
  the simulation.schedules specification updated (see this ADR)
```

---

## Validation

| Scenario | Expected | Mechanism |
|------|------|------|
| `test_ckd_protein` runs 1 step | `dietary_protein = 0.80` | The three-meal pulse (0.27+0.27+0.26) accumulates correctly in `_apply_schedules` |
| `test_glucose_meal` between two meals | `carb_intake = 0.0` | `_apply_schedules`'s pulse mode returns 0 on a step with no event |
| `test_banister` for all 6 steps | Mon-Fri `training_load=70`, Sat=35, Sun=0 | `date_range` plus `days` restrict it correctly |
| Loading a model with `date_range` | The GUI displays the validity period's start and end dates | The frontend parses `date_range` into `validStart`/`validEnd` |
| Running the optimizer | The YAML schedule is suppressed, and optimization controls the variable | `_run_sim` explicitly writes `manual_overrides` |
