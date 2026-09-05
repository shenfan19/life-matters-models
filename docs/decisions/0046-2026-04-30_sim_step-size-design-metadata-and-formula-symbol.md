# ADR 0046 - The Final Step-Size Design: `metadata.step_size` Plus the Formula Symbol `step`
**Date**: 2026-04-30
**Status**: Implemented

---

## Background

The original design placed the step size (`step`) and its unit (`step_unit`) inside the `simulation` block, causing three problems:

1. Step size lock-in: a formula's coefficients implicitly assume a step size, and changing the step size breaks the numbers.
2. YAML ambiguity: `step: 10, step_unit: minute` is not intuitive enough for a modeler.
3. Time-point precision: the larger the step size, the less precise a schedule pulse's window-hit test becomes.

The decisions below solve all three problems.

---

## Decision

### Decision 1: a two-level metadata.step_size structure

```yaml
metadata:
  name: my_model
  step_size:
    value: 1        # the canonical step-size number, recommended to always be 1
    unit: minute    # the time unit a formula's coefficients are calibrated to (defines what step means)
```

- `step_size.unit` = the time unit a formula's coefficients are calibrated to (the model's "clock resolution").
- `step_size.value` = the canonical step size (defaulting to 1, i.e. each step advances by one unit).
- Placed under `metadata` rather than `simulation`, since it is a property of the model's definition, not of the run configuration.

Supersedes: `simulation.step` plus `simulation.step_unit` (removed from the simulation block).

### Decision 2: use the symbol `step` uniformly in formulas

A rate-type formula (a state variable update) must be multiplied by `step`:

```yaml
# Correct: a rate type (a sustained effect per time unit)
dynamics:
  insight:    insight + 0.069 * cognitive_efficiency * step
  nutrition:  max(0, nutrition - 0.010 * step)

# Correct: an instantaneous type (an input pulse, not multiplied)
dynamics:
  stomach_carbs: stomach_carbs + carb_intake
```

`step_size` and `dt` are kept as backward-compatible aliases, with all three carrying the same value.

The Unicode option (`ΔT`) was rejected: `Δ` (U+0394) and `∆` (U+2206) are visually indistinguishable, easily causing a silent bug. The ASCII `step` is completely unambiguous.

### Decision 3: the complete rule for the three variable types

| Variable type | Formula role | Multiplied by step? |
|---------|---------|-----------|
| `state` | A rate formula (continuous dynamics) | Must |
| `input` | An instantaneous pulse (a pulse-driven quantity) | Not multiplied |
| `parameter` | A multiplicative coefficient | Not applicable (it is itself a coefficient) |

### Decision 4: the simulation block keeps only the start/end times and output

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2026-01-04"
  output_variables: [blood_glucose, stomach_carbs]
  schedules: {...}
  # step and step_unit have been removed
```

### Decision 5: a GUI coarsening control

- The toolbar shows the canonical step size (from `metadata.step_size`, a read-only hint).
- A coarsening-multiplier selector: 1x / 2x / 5x / 10x / 30x / 60x.
- The actual run step size = `step_size.value x the coarsening multiplier`.
- `step` in a formula equals the actual run step size, so formulas adapt automatically.

### Decision 6: a multi-file merge constraint

Every imported component file must share the same `metadata.step_size.unit`. The loader warns on a mismatch for now, to be changed to an error later.

---

## Result

```
sim_engine/src/model_structure/loader.py
  reads metadata.step_size.{value,unit} first; falls back to simulation.step/step_unit for backward compatibility

sim_engine/src/model_structure/simulation.py
  symtable['step'] = step_size  (new; step_size/dt kept as aliases)

sim_gui/src/components/Simulator.tsx
  on model load: reads the default step size from metadata.step_size
  toolbar: shows the canonical step size plus a coarsening-multiplier selector (1x to 60x)

Every YAML file under models/
  metadata gains step_size: {value: 1, unit: <the former step_unit>}
  simulation's step and step_unit fields removed
  step_size in formulas replaced with step

docs/model_design.md
  Schema updated: the metadata.step_size two-level structure
  The Euler section: examples switched to the step symbol
  New: a table of variable type versus whether it is multiplied by step
```
