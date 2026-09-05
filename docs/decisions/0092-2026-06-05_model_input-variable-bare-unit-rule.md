# ADR 0092 - The Unit Convention for type: input Variables: a Bare Unit (an Event Quantity), Rate Units Prohibited
**Date**: 2026-06-05
**Status**: Implemented

---

## Background

ADR 0044 established `simulation.schedules`'s pulse execution mode: a value is written when a schedule triggers, and every other step is automatically 0. ADR 0046 established the formula step-size rule: a formula for an input pulse variable is not multiplied by `step`.

But neither ADR made clear what should be written into a `type: input` variable's `unit` field. In practice, a large number of semantic errors appeared: modelers wrote an `input`'s unit as a rate unit (`mg/day`, `kcal/day`, `g/kg/day`, `hours/day`, `kg/week`, `MET-hours/day`), causing:

1. A semantic contradiction: a pulse is a one-time event, while `/day` implies a sustained rate, and the two contradict each other.
2. Optimizer incompatibility: T3 (day-of-week combination optimization) and T4 (date-range optimization) let the same variable trigger at different frequencies or within different time windows. When the unit contains `/day`, the decoded "per-event quantity" cannot keep a consistent meaning across a variable-frequency scenario; a user triggering Monday through Friday versus triggering every other day gets the same single-trigger value, but the rate meaning is completely different.
3. Formula-error propagation: when the unit contains a rate, modelers tend to add `* step` in the formula, conflicting with ADR 0046's rule that "an input is not multiplied by step," causing numeric instability when the step size changes.

---

## Discussion

### The essential semantics of input

Each time a schedule triggers, it delivers one event, and the physical quantity of an event is one-time:

- Taking 50 mg of aspirin: 50 mg (a single-trigger amount)
- Consuming 500 kcal: 500 kcal (a single-trigger amount)
- Reducing body weight by 0.5 kg: 0.5 kg (the target change per trigger)

Trigger frequency (daily, every other day, three times a week) is controlled by the `days` field and the T3 optimizer, unrelated to the variable's own `unit`. The unit only describes "the physical quantity delivered by a single pulse" and should not carry a time denominator.

### Rate semantics belong to parameter

A genuinely continuous rate process (an intravenous infusion rate in `mg/hour`, a basal metabolic expenditure in `kcal/day`, a natural recovery rate in `1/day`) should be modeled as a `parameter`, integrated in the formula as `parameter x step`:

```yaml
# Correct: a continuous rate, a parameter plus * step
infusion_rate:
  type: parameter
  value: 10.0
  unit: mg/hour

formulas:
  plasma_drug:
    dynamics:
      plasma_drug: plasma_drug + infusion_rate * step - clearance_rate * plasma_drug * step
```

```yaml
# Correct: a discrete event, an input with a bare unit, the formula not multiplied by step
oral_dose:
  type: input
  value: 0.0
  unit: mg

formulas:
  plasma_drug:
    dynamics:
      plasma_drug: plasma_drug + oral_dose - clearance_rate * plasma_drug * step
```

### Handling a literature rate reference value

A rate reference given by the literature (such as "100 mg per day") does not go into `unit`; it is recorded in the `reference` or `description` field as background:

```yaml
aspirin_dose:
  type: input
  value: 100.0
  unit: mg                              # a bare unit: a single-trigger amount
  reference: "Antithrombotic Trialists' Collaboration (2002). Literature dose: 100 mg/day"
```

---

## Decision

### Decision 1: an input variable's unit field may only be a bare unit

A `type: input` variable's `unit` field describes the physical quantity delivered by a single pulse trigger, and must always use a bare unit (no time denominator):

| Correct | Prohibited |
|---------|---------|
| `mg`, `g`, `kcal`, `kg`, `MET-h`, `sessions` | ~~`mg/day`, `g/kg/day`, `kcal/day`, `kg/week`, `MET-hours/day`, `hours/day`~~ |

### Decision 2: an input variable's formula is not multiplied by step

Already established by ADR 0046, restated here: whenever a `type: input` variable appears on the right-hand side of a formula, it must not be multiplied by `step`. Only a parameter-driven rate term is multiplied by `step`.

The split-out form for a mixed case:

```yaml
# Correct: the input term stands alone, the parameter term keeps * step
gfr_decline:
  dynamics:
    GFR: GFR - alpha * max(0, dietary_protein - 0.6) * GFR - beta0 * GFR * step
#          ^ the input term (dietary_protein) is not multiplied by step
#                                                    ^ the parameter term (beta0) is multiplied by step
```

### Decision 3: a literature rate reference value goes into reference or description

A rate statement from the literature (such as "500 mg/day" or "2 mg/kg/day") is recorded as citation background in the `reference` or `description` field, not in `unit`:

```yaml
dietary_protein:
  type: input
  value: 0.8
  unit: g/kg                   # a bare unit: a single meal's amount (g per kg body weight)
  description: "Protein intake per meal (a single-trigger amount). Literature reference: 0.6-0.8 g/kg/day recommended for chronic kidney disease"
  reference: "KDIGO 2012 CKD guidelines"
```

### Decision 4: a continuous rate process is modeled as a parameter

Any rate effect that needs to accumulate continuously at every time step (natural recovery, waning, basal expenditure, etc.) is modeled as `type: parameter`, with a unit carrying a time denominator (`1/day`, `mg/hour`, etc.), multiplied by `step` in the formula:

```yaml
clearance_rate:
  type: parameter
  value: 0.198
  unit: 1/hour
```

---

## Affected Files

The `type: input` variables in the following files have been corrected under this rule (2026-06-05):

- `models/references/medical/disease/chronic/hypertension_gout_2026.yaml`
- `models/references/medical/disease/chronic/stress_health_2026.yaml`
- `models/references/medical/medicine/preventive/vaccine_herd_immunity_2026.yaml`
- `models/references/medical/fitness/racket/badminton_2026.yaml`
- `models/references/social/economy/labor/migrant_labor_exploitation_2026.yaml`
- `models/papers/sodium_lifestyle_bp.yaml`
- `models/papers/s2/ckd_protein_a4_s2.yaml`
- `models/papers/s2/hypertension_gout_a5_s2.yaml`
- `models/papers/s2/ibs_diet_a9_s2.yaml`
- `models/papers/s2/masld_insulin_a7_s2.yaml`
- `models/papers/s4/smoking_stress_a6_s4.yaml`
- `models/test/test_plans.yaml`
- `models/test/test_opt_t4.yaml`
- `models/test/test_opt_results.yaml`

`docs/model.md`'s "Unit Convention for Input Variables" section has been updated accordingly.

---

## Relationship to Existing ADRs

| ADR | Content | Relationship to this ADR |
|-----|------|----------------|
| 0044 | Schedule as an input subtype, pulse mode | Establishes the pulse-semantics foundation |
| 0046 | The step-symbol convention, input not multiplied by step | Establishes the formula rule |
| **0092** | **An input's unit must be a bare unit; rate belongs to parameter** | Rounds out the unit convention on top of 0044/0046 |
