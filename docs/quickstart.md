# LM format Quickstart

> Goal: write and run your first LM format model in 30 minutes.
> Prerequisite: the ability to read clinical literature; no programming background needed.

---

## Core Concepts (3 Variable Types)

An LM format model has only three building blocks:

| Type        | Meaning                                    | Analogy                                      |
| ----------- | ------------------------------------------ | -------------------------------------------- |
| `state`     | A metric that changes over time            | A patient's lab value                        |
| `input`     | An intervention (a drug, a diet, exercise) | A doctor's order                             |
| `parameter` | A fixed mechanism coefficient              | A regression coefficient from the literature |

Equations (`equations`) describe how these variables affect each other. That is all there is to it.

---

## Your First Model: a Hypertensive Patient Taking a Blood-Pressure Drug

### 1. The Simplest Version (Runnable)

```yaml
metadata:
  name: hypertension_intro
  description:
    brief: "A demonstration of a blood-pressure drug's effect, an LM format introductory example"

variables:
  med_dose:
    type: input
    value: 0.0
    unit: mg
    description: "The daily dose of a blood-pressure drug (amlodipine-equivalent)"
    reference: "An example value, not a clinical dose"

  SBP:
    type: state
    value: 160.0
    unit: mmHg
    description: "Systolic blood pressure"
    reference: "The initial value represents typical uncontrolled hypertension"

  bp_sensitivity:
    type: parameter
    value: 0.08
    unit: mmHg/mg/day
    description: "The average drop in blood pressure per mg of dose per day"
    reference: "Law et al. (2009) BMJ 338:b1665"

equations:
  bp_daily_change:
    description: "The blood-pressure drug's linear effect (a simplified model)"
    step_unit: day          # the time unit of step in this equation (required)
    dynamics:
      SBP: SBP - bp_sensitivity * med_dose * step

simulation:
  step_size:                # the simulation's execution step size (required)
    value: 1
    unit: day
  start_date: "2026-01-01"
  end_date:   "2026-06-30"
  plans:
    - id: default
      regimens:
        - variable: med_dose
          time_start: "08:00"
          value: 5.0
          label: "5mg taken in the morning"
```

Save this YAML as any `.yaml` file and load it in the Life Matters interface to run it. The output is SBP's curve over time.

---

### 2. Adding a Constraint: Stop the Drug if SBP Gets Too Low

Add a condition inside `equations`:

```yaml
equations:
  bp_daily_change:
    condition: "SBP > 90"          # only takes effect when systolic pressure is above 90 mmHg
    dynamics:
      SBP: SBP - bp_sensitivity * med_dose * step
```

`condition` is an ordinary mathematical expression and can reference any variable.

---

### 3. Adding Optimization: Let the Software Find the Best Dose for You

```yaml
optimization:
  method: nsga2
  objectives:
    - variable: SBP
      metric: final
      direction: minimize
  startpoint:
    regimens:
      - variable: med_dose
        time_start: "08:00"
        label: "Morning dose"
        optimize:
          value: [2.5, 10.0]          # search range: 2.5 to 10 mg
```

Running this produces a Pareto front, the trade-off curve of SBP's final value across different doses.

---

## Variable Type Quick Reference

**When to use `state`**
A metric that evolves over time, where the evolution itself is the core of what you are modeling, such as blood pressure, blood glucose, creatinine clearance, or body weight.

**When to use `input`**
A behavior the patient or a clinician can adjust, such as a dose, a meal's content, or exercise duration. These are exactly the variables the optimizer searches over.

**When to use `parameter`**
A fixed coefficient given by the literature, such as a regression slope, a rate constant, or a population mean. It does not change over time and does not take part in optimization.

---

## Two Common Mistakes

Mistake 1: forgetting to multiply by `step` in a state update

```yaml
# Wrong: the per-step drop is then independent of step size, so changing step size changes the result
dynamics:
  SBP: SBP - bp_sensitivity * med_dose

# Correct
dynamics:
  SBP: SBP - bp_sensitivity * med_dose * step
```

Mistake 2: multiplying an input pulse by `step`

```yaml
# Wrong: a pill's dose does not scale with step size; 5mg is 5mg
dynamics:
  stomach_drug: stomach_drug + med_dose * step

# Correct
dynamics:
  stomach_drug: stomach_drug + med_dose
```

The rule: a `state` evolving continuously gets multiplied by `step`; an `input` delivered instantaneously does not.

---

## Where to Go Next

| Goal | Where to look |
|------|---------|
| The complete field specification | `docs/LM_format_1.0.md` |
| The modeling practice guide | `docs/authoring/README.md` |
| Combining multiple models (import) | `docs/authoring/imports_and_organization.md` |
| Every optimizer parameter | `docs/authoring/regimens_and_optimization.md` |
| Existing runnable models for reference | the `models/papers/` directory |
| Architecture decision background | the `docs/decisions/` directory |
