# Variables, Equations, and Evidence Types

## Variable Types (3) and In-Place evidence_type Conversion

Under the `variables:` block, `type` only accepts 3 values:

| Type | What the engine reads | What the modeler fills in | Purpose | Optimization ownership |
|------|---------|----------|------|---------|
| `state` | `value` (updated over time) | Initial value | A state variable that evolves over time | — |
| `input` | `value` (user-adjustable) | A control quantity | A user intervention quantity (behavior, dose) | **Outer-loop opt (Simulator)** |
| `parameter` | `value` (constant; sampled once per run in MC mode) | A dynamics coefficient or distribution expression | A mechanism coefficient that feeds directly into an equation (a PK rate, an equation slope, Bergman's p1/p2/p3, etc.); its value is determined by fitting the inner-loop optimizer to literature data. `value` may be written as a distribution such as `normal(μ, σ)` to represent individual variation; a deterministic run uses the mean, and MC mode samples once per run. | **Inner-loop opt (Modeller, not yet implemented)** |

> `probability_constant` has been retired: incidence, case-fatality, and similar probability values are now uniformly expressed as `evidence_type: ir`. A `probability_params:` section in an existing YAML file still parses, and the loader maps it automatically.

`evidence_type` is not a fourth `type`, but an optional field on a `variables:` entry (formerly a separate top-level `evidence:` section, later merged into `variables:`, since evidence is essentially still a variable and giving it its own section only created two parallel naming spaces and confusion; see ADR 0137, which supersedes ADR 0040's top-level-section design). An effect size given directly by the literature (OR/HR/RR/Cohen's d, etc.) is declared as the `evidence_type` field of a `variables:` entry, with `value` holding the raw literature figure and `type` required to be `parameter`; at load time the loader converts `value` in place into a coefficient ready for use in an equation, under the same name with no suffix, so `equations`/`dynamics` can reference that name directly, with no need to track a derived name such as an `_effective` variant.

After conversion, the variable additionally carries two provenance fields (not declared in the YAML, filled in automatically by the loader, for lookup and debugging only):

| Field | Meaning |
|------|------|
| `evidence_type` | The type of the original effect size (such as `rr`/`or`/`hr`); `None` for a parameter that is not sourced from evidence |
| `evidence_raw_value` | The original literature figure before conversion (such as OR=1.65), kept separate from the converted `value` for review and traceability |

No separate `VariableType.evidence` was created: a conversion result is merged into the `parameter` type and marked by the two fields above, rather than introducing a fourth type. The reason is that the inner-loop optimizer (Modeller) has not yet been implemented, so no code path currently auto-tunes a `parameter`, and there is no need to isolate types just to prevent mis-optimization; once the Modeller exists, it only needs to skip any parameter where `evidence_type is not None`, with no need to preemptively expand the type set for an optimizer that does not exist yet. This is also why `evidence_type` was not merged with `type` (state/input/parameter, the variable's role in the dynamics) into a single compound string field such as `type: evidence-rr`: role and "the statistical effect-size type of the data source" are two orthogonal dimensions, and merging them would inflate the `type` enum from 3 values to 11 and bind two separate concerns into one field, degrading later role-based lookups, such as finding every parameter, from an equality match to a prefix match. An entry that declares `evidence_type` must have `type: parameter`, or the loader raises an error and rejects it.

Rule of thumb for `parameter` versus `evidence_type`:
- The literature gives you a number that can go directly into an equation, but it comes from a mathematical fit rather than a direct measurement, such as a Bergman-model coefficient: use a plain `parameter` (no `evidence_type` declared, left to the Modeller's inner-loop optimization to calibrate).
- The literature gives you a raw statistical effect size (OR=1.65, HR=0.82, d=0.68, ke=0.198 h⁻¹): use `parameter` plus `evidence_type` (the loader converts it automatically).

Current capability boundary: the GUI has no dedicated visualization or edit form for the `evidence_type` field yet, so modelers need to edit the YAML directly; the GUI's general-purpose variables/equations editor has not been implemented yet, and the evidence form will be filled in once that editor exists.

Automatic wiring into dynamics (`applies_to`, optional): once a coefficient has been converted, wiring it into some state variable's dynamics equation is still left to the modeler to write by hand by default. Of the 8 subtypes, `ir`/`ard`/`hr`/`rr`/`or` have exactly one unambiguous way to be wired in, namely "use the converted coefficient as a rate and accumulate it into some target state," so declaring the following fields makes the loader generate the corresponding dynamics automatically:

| Field | Applicable subtypes | Meaning |
|------|----------|------|
| `applies_to` | ir/ard/hr/rr/or | The name of the target state variable (must already be declared in `variables:`); declaring this triggers automatic generation |
| `step_unit` | ir/ard/hr/rr/or | The step-size unit used by the generated dynamics, which must be `minute`/`hour`/`day` (the same constraint as `equations.step_unit`) |
| `rate_unit` | ir/ard declare it themselves; hr/rr/or read it from the ir/ard entry `baseline_ref` points to | The rate's natural time unit (one of `minute`/`hour`/`day`/`week`/`month`/`year`); its ratio to `step_unit` is computed into a coefficient written into the generated expression, independent of `Equation.step_unit` expressing year/week/month |
| `baseline_ref` | hr (already existed), rr/or (newly added) | Must point to an `evidence_type: ir/ard` entry under `variables:` in the same file; an rr/or conversion result is only a ratio and needs this baseline to generate a "baseline times ratio" rate |

```yaml
variables:
  baseline_cvd_ir:
    type: parameter
    evidence_type: ir
    value: 0.012
    unit: prob/year
    rate_unit: year              # the rate's natural time unit
    applies_to: cvd_risk_baseline # auto-generated: cvd_risk_baseline += baseline_cvd_ir * (day/year) * step
    step_unit: day

  smoking_cvd_rr:
    type: parameter
    evidence_type: rr
    value: 2.5
    baseline_ref: baseline_cvd_ir  # rr itself is only a ratio and needs the baseline to generate a rate
    applies_to: cvd_risk_smoker
    step_unit: day
```

Constraints (to avoid introducing an implicit scientific assumption, these fail loudly rather than silently ignoring the problem):
- Declaring `applies_to` on `cohens_d`/`beta`/`pk` raises an error directly, since how these three are wired in is itself a modeling judgment (the transitional form, the regression structure, and the PK model structure are not unique), so automatic generation is never supported for them and dynamics must always be written by hand.
- The same `applies_to` target declared by two or more evidence entries at once raises an error, since how multiple risk factors combine, multiplicatively under a proportional-hazards assumption or additively under a competing-risks model, is itself a contested epidemiological methodology question that the engine will not choose on the modeler's behalf; remove `applies_to` and write the dynamics by hand instead.
- `applies_to` is only meaningful on an entry that declares `evidence_type`; declaring `applies_to` without `evidence_type` raises an error, to prevent a plain parameter from misusing this field.
- Not declaring `applies_to` leaves behavior completely unaffected, and dynamics continue to be written by hand as before; this is a purely additive field.

See ADR 0040's "Implementation Record" (the original design of the top-level `evidence:` section) and ADR 0137 (the subsequent decision to merge it into `variables:`) for the design derivation.

Unit convention for `input` variables (a quantity of an event, the single rule):

The LM engine executes `input` variables in sustained mode (ADR 0127): a step that falls inside the effective window is written as `value/N_steps`, a step outside the window is automatically 0, and the accumulated contribution always equals `value`. The `unit` field describes the total physical quantity delivered within the window, always as a bare unit with no time denominator. An `input` equation adds or subtracts it directly, without multiplying by `step`:

| Correct | Prohibited | Reason |
|---------|---------|------|
| `mg`, `g`, `kcal`, `kg`, `MET-h`, `sessions` | ~~`mg/day`, `g/kg/day`, `kcal/day`, `kg/week`~~ | The schedule's trigger frequency is controlled by `days`, and `/day` is incompatible with the T3/T4 optimizer |

A rate reference value given by the literature, such as "500 mg/day" or "2 mg/kg/day," is recorded in the `reference` or `description` field, not in `unit`. A genuinely continuous rate process, such as an intravenous infusion rate, is modeled as a `parameter` (with a unit such as `1/day` or `1/min`) integrated in the equation together with `× step`; an `input` variable's equation does not use `× step`.

> Examples: `aspirin_dose = 50 mg` (a morning dose, with the literature's "100 mg/day" recorded in `reference` and `unit` written simply as `mg`); `caloric_deficit = 500 kcal` (triggered daily, not written as `kcal/day`); `weight_loss_weekly = 0.5 kg` (triggered every Monday, not written as `kg/week`).

The 8 values of `evidence_type`: `rr` (relative risk), `or` (odds ratio, needs `baseline_prevalence`), `hr` (hazard ratio, needs `baseline_ref`), `ard` (absolute risk difference), `cohens_d` (effect size, needs `population_sd`), `ir` (incidence/mortality rate), `beta` (regression coefficient), `pk` (PK/PD parameter).

The authoritative version of the conversion equations is not in this file; it is in the `life-matters-reference-engine` repository's `docs/evidence/conversion.md` (which covers the `evidence_type`/`evidence_raw_value` provenance fields and known implementation details, such as `hr`'s `baseline_ref` not validating the target type on the base conversion path). The validation order and generated-expression templates for `applies_to`'s automatic wiring into dynamics are in `applies_to.md` in the same directory. This file only maintains which YAML fields a modeler needs to fill in; the equations themselves change with the loader's implementation, and to avoid maintaining and drifting in two places, treat `conversion.md`/`applies_to.md` (matching the actual loader.py code) as authoritative whenever the two disagree.

---

## Medical Evidence Types and Variable Mapping

A variable that declares `evidence_type` is converted automatically by the loader at load time, and the Simulator only ever sees the converted value, under the same name it was declared with, with no suffix. The complete conversion logic for the 8 subtypes is in the `life-matters-reference-engine` repository's `docs/evidence/conversion.md`; a YAML example is in the schema below, and the decision background is in `decisions/0040` (the top-level-section design) and `decisions/0137` (merging it into `variables:`).

Prevalence is set directly as the initial `value` of the corresponding `state` variable and does not need a separate `evidence_type` declaration.

---

## Variable and Equation Data Conventions

Every `variables` and `equations` entry across all YAML files must follow these rules:

1. **`description` is required**: a concise statement of the variable's or equation's physical or medical meaning.
2. **`reference` is required**: every value (`value`), range (`bounds`), and dynamics equation (`dynamics`) must be annotated with a data source. The format is not fixed, but it must carry enough information, a DOI, a PMID, a short citation, or a URL, for a reader to locate the original publication within 30 seconds. When no source is available yet, fill in `["TODO:SOURCE"]` and note the estimation logic in `description`.
3. **`locator` is optional**: paired with `reference`, it points to the value's exact location within the publication (page number, figure, table, equation, section); the format is not fixed but should be specific, such as `"Table 1"`, `"Figure 3"`, `"eq.3"`, `"p.1172"`, or `"§6.2"`. When a specific location is known, prefer writing it into `locator` rather than scattering location information through the `description` text; when the specific location has not yet been verified, fill in `"TODO:LOCATE"` rather than inventing a page or figure number. The GUI concatenates the reference column automatically as `reference (locator)`.
4. **`comments` is optional**: records the reasoning behind a choice made when publications conflict, or the process of a parameter's fine-tuning, and does not replace `description` or `reference`.

---
## Time and Step Size

Step size splits into two independent concepts, each declared in its own field (ADR 0104, ADR 0105):

### equation.step_unit (Required When dynamics Uses step)

`step_unit` is required only when the equation's `dynamics` expression uses `step`, and it declares the time unit that the `step` symbol represents within that equation. A `dynamics:` equation that does not contain `step` (a pure algebraic assignment, for instance) does not need to declare `step_unit`.

```yaml
equations:
  bp_dynamics:
    description: "..."
    step_unit: day        # minute | hour | day (required when dynamics uses step)
    dynamics:
      systolic_bp: "systolic_bp + (...) * step"

  performance_calc:
    description: "A static calculation, no step, no step_unit needed"
    dynamics:
      performance: p0 + fitness - fatigue
```

`step_unit` is a property of the equation, reflecting the time resolution assumed when its coefficients were calibrated. When importing across modules, each equation carries its own `step_unit`, and the engine converts the numeric value of `step` accordingly.

### simulation.step_size (Required)

The simulation's execution step size, independent of an equation's `step_unit`:

```yaml
simulation:
  step_size:
    value: 1
    unit: day             # minute | hour | day
```

`optimization.step_size` uses the same format and is optional, defaulting to `simulation.step_size` when omitted.

### Symbols Inside an Equation

| Symbol | Meaning | Note |
|------|------|------|
| `step` | The current equation's step size (in units of `equation.step_unit`) | **The only standard symbol** |
| `t` / `time` | The current simulation time (in units of `equation.step_unit`) | |
| ~~`step_size`~~ / ~~`dt`~~ | Same as `step` | **Deprecated**, prohibited in new equations; the validator raises an error if it detects one |

How `step` is computed: `step = simulation.step_size / equation.step_unit` (converted to the same time unit before dividing).

| Example | `simulation.step_size` | `equation.step_unit` | The value of `step` inside the equation |
|------|----------------------|---------------------|----------------|
| Same unit | 1 day | day | 1 |
| Coarse step size x fine unit | 1 day | hour | 24 (each step integrates 24 hour-units) |
| Fine step size x coarse unit | 1 hour | day | 1/24 (each step integrates only 1/24 of a day) |

---

## Equation Step-Size Rules

Whether to multiply by `step` is determined by the variable type:

| Variable type | Equation type | Multiply by step? | Reason |
|---------|---------|-----------|------|
| `state` | A rate (continuous dynamics) | **Must multiply** | The effect is proportional to elapsed time |
| `input` | A pulse (regimen-driven) | **Do not multiply** | A one-time quantity, independent of step size |
| `parameter` | A multiplicative coefficient | Not applicable | It is itself a coefficient |

```yaml
# Correct, rate-type: a state update must multiply by step
dynamics:
  insight:   insight + 0.069 * cognitive_efficiency * step
  nutrition: max(0, nutrition - 0.010 * step)

# Correct, instantaneous-type: an input pulse does not multiply by step
dynamics:
  stomach_carbs: stomach_carbs + carb_intake
```

This rule is most easily forgotten in the combined pattern of "pulse input plus a decaying state." The literature's continuous ODE for this is commonly written `dA/dt = g*w(t) - k*A`, and copying it directly into an Euler update produces `A: A + (g*w - k*A) * step`, which is syntactically valid but places `w` (the pulse input) and `k*A` (the decay term) inside the same parenthesis and multiplies both by `step`, exactly the usage the table above marks as prohibited; because the two terms sit together and look identical to the textbook's continuous equation, it is easy to copy this pattern without noticing.

Why this cannot be written this way is not merely a rule specific to this engine: `w(t)` in the original literature context usually represents a once-daily training or dosing event, and the more mathematically accurate description is a series of discrete impulses (a sum of Dirac deltas), not a truly continuous integrable function. The standard way to numerically discretize an ODE with such an impulsive forcing term is to add the pulse term straight into the state as an instantaneous jump, and only apply the `Δt` scaling to a genuinely continuous decay or recovery term; this is the general numerical-methods treatment of impulsive forcing, and it holds regardless of the engine or language used, not an extra rule specific to LM format. The test is simple: if a term's value is determined by a single delivery from `regimens:` (either `delivery: total` or `delivery: level`), it does not get multiplied by `step`, whether or not it is written on the same line or inside the same parenthesis as another, genuinely decaying term.

Models known to have once violated this rule, since fixed (found in a full-library sweep on 2026-08-12, all now fixed):

| File | Equation | Symptom |
|---|---|---|
| `banister_validation.yaml` | `fitness_dynamics`/`fatigue_dynamics` | Race-day performance drifted with `step_size`, pinning against the variable's upper bound at a 6-hour step |
| `hypertension_gout_sim.yaml` | `thiazide_level_dynamics`/`allopurinol_level_dynamics` | `uric_acid` pinned against its upper bound at a 6-hour step |
| `diuretic_tradeoff_sim.yaml` | `thiazide_level_dynamics` | Same as above |
| `postpartum_recovery_sim.yaml` | `caloric_intake_smoothing` | The same pattern, with a comment stating "reused an already-validated pattern," but the source being reused was itself wrong |
| `sodium_lifestyle_bp_sim.yaml` | `net_sodium_balance_dynamics` | Same as above |
| `masld_insulin_sim.yaml` | The `dietary_glycemic_load` term inside `liver_fat_dynamics` | Within the same model, the two parallel input variables `caloric_deficit`/`exercise_met_min` were both written correctly, and only this term was wrong, even though all three declare their regimen the same way |

There is currently no automatic detection: the validator does not check whether this rule has been violated, for the same reason ADR 0121 declines to auto-convert parameter units. Whether an input variable should be multiplied by `step` in a given equation requires understanding that equation's physical structure, which is beyond the design principle that "a formula is a mathematical expression, and the engine does not perform semantic inference." After writing an equation with a "pulse plus decaying state" pattern, it is worth running a step-size grid check yourself, running the same model with several different `simulation.step_size` values such as 1h/6h/15min and checking whether the final value converges to the same number, rather than discovering the problem only right before a paper submission.

### Cross-step_unit Parameter Conversion: Linear Division Versus a Root (ADR 0121)

A rate parameter from the literature is often given per day, such as "half-life 4 to 5 days" or "declines 1.5 mL/min per year," while the equation it sits in may have a finer `step_unit` such as `hour`. The `step_unit` mechanism only converts the equation's own `step` symbol; it does not convert the literal parameter values embedded in the dynamics expression. That conversion is the modeler's responsibility, and getting it wrong raises no error; it silently scales the effect up or down by some factor, typically a 24x amplification going from day to hour.

Before converting, first determine which category the dynamics term the parameter sits in belongs to, since the two categories convert differently:

| Category | Term form | Conversion method |
|------|---------|---------|
| **A: a state-independent flux term** | `X: X + rate * f(other variables) * step` (the coefficient does not depend on the same state variable being updated) | **Exact linear**: the daily rate divided by 24 gives the hourly rate |
| **B: a self-exponential decay or recovery term** | `X: X - k * (X - target) * step` (the coefficient multiplies the difference between the state itself and a target value, where target can be 0) | **A root**: $k_{hour}=1-(1-k_{day})^{1/24}$; dividing linearly by 24 is only an approximation valid when $k_{day}$ is small (for example, at $k_{day}=0.15$ the deviation is about 8%) |

Category B is the Euler discretization of the first-order linear ODE $dX/dt=-k(X-\text{target})$; the compound effect of 24 hourly steps is multiplicative ($(1-k_{hour})^{24}$), not additive, so it cannot simply be divided by 24. The two computation methods cost exactly the same, a single exponentiation or a single division, computed once when the model loads, so category B must use the root equation and a linear approximation is not acceptable.

Do not "fix" a category B parameter by simply changing the whole equation's `step_unit`. `step_unit` is a property of the entire equation, and if the same equation mixes in other terms already correctly calibrated to the current `step_unit`, a common situation, changing `step_unit` incorrectly dilutes those terms as well. The correct approach is to change only that parameter's own `value` (updating `unit` and `description` together to leave a trace) while keeping the equation's `step_unit` unchanged. See ADR 0121 for details.

---

## Equation Execution Order (priority)

Within each step, equations are sorted by `priority` in descending order and executed in that order (a larger number executes first; the default when undeclared is 0). The sort is global and one-time: equations are ordered by their overall `priority`, and within a single equation, evaluation follows the fixed order of `condition` then `dynamics`.

Within-step ordering has write-through semantics, not a "snapshot": each equation's newly computed value is written back to the model's variables immediately (including `bounds` clipping), and becomes visible right away to any equation that still executes later within the same step. That is, an equation with a larger `priority` value executes first, and its written-back result is read by equations with a smaller `priority` value later in the same step, not the previous step's stale value. If equation B needs to read equation A's latest result from the same step, give A a larger `priority` than B.

```yaml
equations:
  feed_intake:          # Runs first: adds the feed volume to the stomach and handles overflow
    priority: 10
    dynamics:
      stomach_volume: "stomach_volume + intake - max(0, stomach_volume + intake - capacity)"

  gastric_emptying:     # Runs later: reads the stomach_volume feed_intake already updated this step
    priority: 0
    dynamics:
      stomach_volume: "stomach_volume - emptying_rate * stomach_volume * step"
```

---

## Euler Discrete Integration (a Permanent Decision)

This framework permanently uses a uniform, explicit Euler discretization, writing out the next value directly, and does not introduce a higher-order integrator such as RK4.

```yaml
dynamics:
  blood_glucose: blood_glucose + (uptake - utilization) * step
  position:      position + velocity * step
  velocity:      velocity + (force - damping * velocity) * step
```

Reasoning: physiological and social model parameters carry ±10-50% uncertainty, far larger than Euler's truncation error; discrete events such as a meal or a dose already defeat a higher-order integrator's precision advantage; and an explicit form is what you see is what you get.

---
