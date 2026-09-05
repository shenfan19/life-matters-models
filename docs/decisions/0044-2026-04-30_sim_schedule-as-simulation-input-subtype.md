# ADR 0044 - simulation.schedules: Time-Driven Input Belongs Under the simulation Block, as an input Subtype; the GUI Auto-Prefills a Regimen
**Date**: 2026-04-30
**Status**: Implemented

---

## Background

Test scenarios such as `test_glucose_meal.yaml` introduced a time-driven input variable (three meal times driving blood-glucose fluctuation), requiring a decision on where "a time-series input" belongs in the YAML, and how the GUI should present it.

Problems that surfaced:

1. The original implementation made `schedules:` a top-level key, alongside `variables:` and `formulas:`.
2. The GUI read `content.schedules` (the wrong path), and rendered it as a read-only block, completely disconnected from the same-named `type: input` variable under `variables`; a user looking at the Inputs panel saw only an empty input box for `carb_intake = 0.0`, with the schedule data effectively invisible.
3. A risk of mixing the top-level `schedules` with the optimizer's fields: if optimization needs its own time series, a naming conflict would result.

---

## Discussion

### Where it belongs

`schedules` describes "how a simulation run drives some input quantity's change over time," which is simulation configuration, not a variable definition and not a dynamics formula. It should therefore belong under the `simulation` block, alongside `start_date`, `step`, and the like.

If the Optimizer needs a time series (such as an optimal dosing schedule), it can define its own inside the `optimization` block, with the two namespaces kept separate.

### Its semantic role

`schedules` is essentially a subtype of `type: input`; it states that "some input variable's value is driven by a time series inside the YAML, rather than manually filled in by the user in the GUI." A schedule's key must therefore be the name of a `type: input` variable already declared in `variables`.

### GUI presentation

The original approach: a read-only block displayed separately, disconnected from the Regimen system.

Problem: the user could not edit it, and it was visually disconnected from "an operable input," easily mistaken for the model having no input at all.

New approach: when a model loads, the points in `simulation.schedules` are automatically converted (`time_seconds % 86400` to `HH:MM`, with repeated multi-day points deduplicated by time of day) and pre-filled into the corresponding variable's Regimen card. The user can edit directly from there, with behavior identical to manually creating a Regimen.

### The zero-value-point rule for a discrete input

Under the Euler discrete-stepping model, an instantaneous `type: input` quantity (a meal amount, a dose) is independent step by step, not a continuously held quantity. So a schedule does not need a `value: 0` "closing point"; only the moments with an actual input need listing.

---

## Decision

### Decision 1: schedules belongs under the simulation block

```yaml
# Correct
simulation:
  start_date: "2026-01-01"
  end_date: "2026-01-04"
  step: 10
  step_unit: minute
  output_variables: [blood_glucose]
  schedules:
    carb_intake:
      interpolation: step
      points:
        - {time: 25200, value: 1.5}
        - {time: 43200, value: 1.8}

# Deprecated: a top-level schedules
schedules:
  carb_intake: ...
```

The engine loader reads from `simulator_data.get('schedules', {})` (`simulator_data` now uniformly points to `data['simulation']`).

### Decision 2: a schedules key must correspond to a type: input variable

A schedule's meaning is "driving some input quantity," so every schedule key must have a corresponding `type: input` declaration in `variables`. This makes the relationship between the two explicit and avoids the silent failure of "a schedule exists but its variable was never declared."

### Decision 3: the GUI auto-prefills a Regimen

When a model loads (a `useEffect` on `selectedModel`):
1. Check `simulation.schedules`.
2. For each input variable with a schedule, convert its points into `RegimenEvent[]` (`time_seconds % 86400` to `HH:MM`, deduplicated by time of day, keeping the first occurrence).
3. Pre-fill them into that variable's Regimen card.

A pre-filled Regimen behaves exactly like a manually created one, and the user can edit it freely.

### Decision 4: a discrete input does not write a zero-value point (a modeling rule)

> An instantaneous `type: input` quantity (a meal amount, a dose, etc.) lists only its nonzero moments in a schedule, with no inserted `value: 0` closing point.

This rule is written into `model_design.md`'s section on `simulation.schedules`.

Exception: a continuous-rate-type variable (such as a sustained infusion's `infusion_rate`, expected to stay nonzero over a stretch of time) may keep a closing point if needed.

### Decision 5: adding a new pulse interpolation mode

To support an instantaneous quantity (a meal, a dose, a one-time pulse event) with no zero-value point needed, a new `interpolation: pulse` mode was added:

- At the start of each step, check whether a schedule event's time falls within the window `[self.time, self.time + step_size_sec)`.
- If it hits, apply that point's value; if it misses (between events), it is automatically 0.
- The `step` / `linear` modes were corrected at the same time: both now return 0 before the first point and after the last point (the original behavior held the first/last point's value, causing a nonzero input right from the start of the simulation).

```yaml
schedules:
  carb_intake:
    interpolation: pulse   # each meal is only in effect at the step it hits; every other step is automatically 0
    points:
      - {time: 25200, value: 50}   # 07:00 breakfast, 50g
      - {time: 43200, value: 80}   # 12:00 lunch, 80g
```

`step_size_sec` is passed into `_apply_schedules(step_size_sec)` through the `step()` method's parameter.

---

## Result

```
sim_engine/src/model_structure/loader.py
  schedules now read from simulator_data.get('schedules') (was data.get('schedules'))

sim_engine/src/model_structure/simulation.py
  _apply_schedules(step_size_sec): added pulse mode; fixed the step/linear first/last-point behavior
  step(): computes step_size_sec and passes it to _apply_schedules

sim_gui/src/components/Simulator.tsx
  the schedules read path: content.simulation.schedules
  useEffect: an input variable with a schedule is auto-prefilled with Regimen events
  removed: the read-only schedules display block

models/scenarios/test/
  test_glucose_meal.yaml   rebuilt: pulse mode, two MC parameters, correct dynamics formulas
  test_daily_life.yaml     schedules moved under the simulation block
  test_schedule.yaml       schedules moved under the simulation block

docs/model_design.md
  Schema updated: simulator to simulation, a new schedules field added
  New: the simulation.schedules specification section (including the discrete-input rule and the pulse explanation)
```
