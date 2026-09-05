# ADR 0104 - A Step-Size Design Refactor: per-formula step_unit + simulation.step_size

**Date**: 2026-06-16
**Status**: Adopted
**Supersedes**: ADR 0046 (the metadata.step_size design)

---

## Background

ADR 0046 placed `step_size` under `metadata` as a whole model's global step size, giving it two roles at once:

1. A formula's semantic unit: the length of time the `step` symbol represents inside a formula.
2. The simulation's execution step size: the amount of time the engine actually advances per step.

In addition, `optimization.step_size` could optionally override the simulation's execution step size for optimization.

Problems:

- `metadata`'s purpose is descriptive fields (name, tags, description); a step size carrying computational semantics does not semantically belong there.
- `optimization.step_size`'s existence already separates the two roles, but the simulation side had no symmetric field, creating an asymmetry.
- A formula's step-size unit is a property of the formula itself (the time scale its coefficients were calibrated at), not a model-level global property; when importing across modules, different formulas within the same run may come from source models with different step sizes, and a global metadata field cannot accurately express this difference.

---

## Decision

Fully separate the two roles into two independent fields:

### 1. `formulas.<name>.step_unit` (required, a string)

```yaml
formulas:
  bp_dynamics:
    description: "..."
    step_unit: day          # minute | hour | day
    dynamics:
      systolic_bp: "systolic_bp + (...) * step"
    priority: 7
```

- Declares the time unit the `step` symbol represents inside this formula.
- Required: enforced by the validator, raising an error if missing.
- No `value` field: a formula's calibration-unit value is always 1 and needs no declaration.
- When importing across modules, each formula carries its own `step_unit`, read directly by the loader with no need to look up the source module's metadata.

### 2. `simulation.step_size` (required, `{value, unit}`)

```yaml
simulation:
  step_size:
    value: 1
    unit: day               # minute | hour | day
  start_date: "YYYY-MM-DD"
  end_date:   "YYYY-MM-DD"
```

- Declares the simulation's execution step size, fully symmetric with `optimization.step_size`.
- Required: enforced by the validator.
- `optimization.step_size` stays optional (defaulting to an override value passed in through the API, when omitted).

### 3. How the `step` value is computed

At each step, the engine converts `simulation.step_size` to seconds (`step_size_sec`) and `formula.step_unit` to seconds as well (`step_unit_sec`), dividing the two to get the `step` value injected into the formula:

```
step = step_size_sec / step_unit_sec
```

Example: `simulation.step_size = 1 day`, `formula.step_unit = hour` gives `step = 86400 / 3600 = 24`.

### 4. `metadata.step_size` removed

- The `step_size` field is removed from metadata.
- Every existing YAML file is migrated: `metadata.step_size` is split into `simulation.step_size` plus a `step_unit` on each formula.

---

## Engine Adaptation

| Module | Change |
|------|------|
| `loader.py` | Reads `simulation.step_size` instead of `metadata.step_size`; for each formula, reads `form_data['step_unit']` and sets `formula.step_unit` and `formula.step_size_sec` |
| `validator.py` | New: a required check for `simulation.step_size`; a required-plus-value-range check for each formula's `step_unit` |
| `base.py` | The `Formula` dataclass gains a `step_unit: Optional[str]` field |
| `optimizer_engine.py` | Unchanged (already reads `optimization.step_size` independently) |
| `simulator_engine.py` | Unchanged (reads the `simulator['step_size']` injected by the loader) |

---

## Cross-Module Import Behavior

Cross-step-size imports used to rely on the `merged_sources['step_sizes']` dict (recording each source module's `metadata.step_size`).

New approach: each formula's `step_unit` field stays with the formula's data after import merging, and the loader reads it directly, with no need for a source-module step_sizes index anymore.

Example: a top model (step = 1 hour) imports a base model (step = 1 day):
- `mass_decay` (from base): `step_unit: day` gives `step_size_sec = 86400`
- `drug_decay` (the top model's own): `step_unit: hour` gives `step_size_sec = 3600`

The engine injects the correct `step` value for each formula independently at each step.

---

## Trade-Offs

Given up: `metadata.step_size` as a global default, which let a user write one fewer field in a single-module model.

Gained:
- Explicit semantics: a formula's step size is the formula's own property, not the model's property.
- Symmetry: sim and opt each independently declare their step size, on equal footing.
- No implicit inheritance: the validator forcibly requires everything explicit, matching LM format's flat-expression principle.
- Self-contained cross-module import: each formula carries its own unit, with no global index needed.
