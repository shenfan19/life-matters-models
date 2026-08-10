---
title: Life Matters Format (LM format) 1.0
version: "1.0"
status: draft
initial_author: Fan Shen
institutional_affiliation_at_initial_release: Sun Yat-sen University, School of Systems Science and Engineering
date: 2026-05-10
---

# Life Matters Format (LM format) 1.0

**LM format is a scholarly model specification. The LM Reference Engine is one implementation of this specification. Future versions of the specification may be maintained by the author and contributors through the public LM format governance process.**

---

## Introduction

Life Matters Format (LM format) is an open YAML specification for defining executable models of individual-level health and social dynamics. LM format enables researchers to express physiological, behavioral, and socio-environmental models as structured, citable, and interoperable YAML files — without requiring programming expertise to author models.

LM format is not a software product. It is a **format standard**, in the tradition of SBML (systems biology markup language), CellML, and COMBINE/OMEX, but targeting a distinct modeling scope:

> **LM format addresses individual-level behavioral intervention modeling** — the space where clinical guidelines, lifestyle behavior, pharmacological schedules, and social environment interact over daily-to-yearly time scales to produce health outcomes.

This niche is not currently served by existing formats:
- SBML and CellML target biochemical network models (molecular to cellular scale).
- COMBINE/OMEX addresses packaging of multi-format model archives.
- PhysiCell and agent-based frameworks address multicellular and population dynamics.
- No existing open format addresses the **calendar-semantic, multi-scale, individual behavioral intervention** space that LM format targets.

---

## Scope

LM format is designed for:

- **Health and social dynamics models** at the individual level (one person, one social context)
- **Multi-scale composition**: models operating at different native time scales (hours, days, weeks, months, years) that share state variables and can be coupled
- **Behavioral intervention inputs**: diet, exercise, medication schedule, social behavior — expressed as time-varying schedules with calendar semantics
- **Optimization targets**: models whose inputs are candidates for regimen optimization
- **Model library sharing**: YAML files that can be contributed to a public library, cited independently, and composed by other researchers

LM format is **not** designed for:
- Molecular dynamics or sub-cellular biochemical networks (use SBML)
- Population-level epidemiological models (use existing ODE/ABM frameworks)
- Multi-organ systems requiring continuous fluid dynamics (use FEA/CFD tools)

### Inclusion Test

Whether a specific candidate topic falls within this scope can be checked against four questions, each mapped to one LM mechanism:

- **`variables`/`evidence`**: does the claim have a transferable numeric value (an effect size, a rate constant, a dose-response coefficient), not only a directional statement ("has an effect," "is associated with")?
- **`equations`**: does the mechanism have an accepted functional form (a differential equation, a regression equation, a kinetic equation) that can be encoded directly, without the modeler inventing structure on the spot?
- **`simulation`**: once encoded, can the mechanism produce a trajectory that evolves over time and can be checked against an independent data point, rather than only a static number?
- **`optimizer`**: does the decision space contain a genuine multi-objective tradeoff (not a single best answer) worth searching with an optimizer?

The first two questions determine whether a topic is worth encoding as an LM file at all; the last two determine what the resulting model can be used for once built. A model whose `variables`/`evidence`/`equations` pass but whose `simulation`/`optimizer` questions do not apply is still a valid LM file — it can only be used for mechanism/engine demonstration, not independent prediction or optimization.

This test is deliberately coarse and topic-agnostic: it says nothing about whether a *specific* candidate model is correct, only whether the topic is a reasonable target for encoding at all. A living inventory of which sub-disciplines currently pass or fail each question is maintained by the LM Reference Library outside this specification (`docs/authoring/methodology.md`), not in the versioned format spec, since that inventory changes with every modeling round and reflects one library's curation practice rather than a property of the format itself.

The four mechanisms behind these questions give LM format its working shorthand, V.E.S.O. — Variables, Equations, Simulation, Optimization — the same four top-level blocks introduced in §1.1 below.

---

## Terms and Definitions

| Term | Definition |
|------|-----------|
| **LM format** | Life Matters format — this specification |
| **LM file** | A YAML file conforming to this specification |
| **Component** | An LM file defining reusable physiological or behavioral sub-models |
| **Scenario** | An LM file that includes a `simulation:` block and is directly runnable |
| **LM-compatible engine** | Any software that can load, validate, and execute LM files according to this specification |
| **LM Reference Engine** | The first LM-compatible engine, authored by Fan Shen at Sun Yat-sen University |
| **K×4 Regimen** | A calendar-semantic input formalism defined in this specification (§6) |
| **Model Library** | A curated collection of LM files, open for contribution and citation |
| **step** | The canonical time step duration as declared in `metadata.step_size` |

---

## 1. File Format

LM files are YAML 1.2 documents. A conforming LM file must have a `metadata:` block and at least one of: `variables:`, `equations:`, `imports:`. All keys are lowercase snake_case.

### 1.1 Top-Level Structure

```yaml
# Top-level keys in an LM file (all optional except metadata)
metadata:       # Required. Identification, versioning, citation, license.
imports:        # Optional. References to other LM files to merge.
variables:      # Optional. state, input, and parameter declarations (including literature effect sizes via evidence_type).
equations:      # Optional. Dynamic equations and instantaneous calculations.
simulation:     # Optional. Makes this file a runnable Scenario.
optimizer:      # Optional. Multi-objective optimization configuration and results.
```

### 1.2 Metadata Block

```yaml
metadata:
  # --- Identity (required) ---
  name: "ckd_protein_muscle"           # Unique identifier, snake_case
  version: "1.0"                        # Semantic version of this model file

  # description accepts a plain string or a structured mapping.
  # Structured form: any subset of keys, plus custom keys, GUI displays all
  # non-empty fields in YAML key order. The LM Reference Library's own
  # convention for papers/ models is problem/method/result/limitations,
  # see docs/authoring/description_writing.md for the reasoning and the
  # recommended way to write each field.
  description:
    problem: "Bidirectional conflict: low protein protects kidneys but accelerates muscle wasting."
    method: "Euler ODE, day-scale, coupled via LM format imports."
    result: "Pareto front over protein intake shows the tradeoff has no dominant point."
    limitations: "Muscle catabolism multiplier is a calibrated fit, not a direct literature value."
  # Plain string form is also valid: description: "CKD protein-muscle tradeoff model."

  # --- Classification (recommended) ---
  tags: [medical, renal, musculoskeletal, chronic_kidney_disease]
  lm_format_version: "1.0"                    # Which LM format version this file targets

  # --- Time scale (required for Scenarios; recommended for Components) ---
  step_size:
    value: 1
    unit: day                           # second | minute | hour | day | week | month | year

  # --- Authorship (recommended) ---
  authors:
    - name: "Fan Shen"
      email: "shenfan@example.edu"      # optional

  # --- Literature sources this model draws on (recommended when parameters come from literature) ---
  # Each entry may be a plain citation string, or an object pairing the citation with what
  # it specifically contributes to this model; see docs/authoring/description_writing.md.
  references:
    - citation: "KDIGO 2020 Clinical Practice Guideline for Diabetes Management in CKD. Kidney Int."
      description: "GFR decline rate and protein restriction threshold."
    - citation: "Bauer J et al. (2013) Sarcopenia in CKD. NDT."
      description: "Muscle loss rate under CKD."

  updated: "2026-05-10"
```

---

## 2. Variables

The `variables:` block declares all quantities in the model. Every variable has a `type` that determines how it is treated by LM-compatible engines.

### 2.1 Variable Types

Variables declared in `variables:` belong to one of three optimization roles:

| Type | Role | Step-multiplied? | Can be optimized? | Example |
|------|------|------------------|--------------------|---------|
| `state` | Time-evolving quantity | Yes (dynamics) | No (it evolves) | `gfr`, `muscle_mass` |
| `input` | Intervention dose or control | No (instantaneous) | Yes (outer loop: Regimen) | `protein_intake`, `exercise_bout` |
| `parameter` | Physiological constant | As coefficient | Yes (inner loop: model fitting) | `gfr_decline_rate` |

Literature-derived effect sizes (RR, OR, HR, Cohen's d, incidence rates, regression coefficients, and PK constants) are declared as `parameter` entries with an additional `evidence_type` field — not a separate variable role. See §2.4.

### 2.2 Variable Declaration

```yaml
variables:
  gfr:
    type: state
    value: 45.0                         # Initial value
    unit: "mL/min/1.73m²"
    bounds: [0, 120]                    # [min, max] — hard physiological limits
    description: "Glomerular filtration rate"
    reference: "KDIGO 2020"

  protein_intake:
    type: input
    value: 0.8                          # Default value when no schedule is provided
    unit: "g/kg"                        # Bare unit: one pulse event quantity; NOT a rate
    bounds: [0.3, 2.5]
    description: "Protein intake per meal event (one pulse trigger). Literature reference: CKD guideline 0.6–0.8 g/kg/day"
    reference: "KDIGO 2020"

  gfr_decline_rate:
    type: parameter
    value: "normal(0.003, 0.0008)"      # Distribution for Monte Carlo
    unit: "fraction/day"
    bounds: [0.0005, 0.008]
    description: "Baseline GFR decline rate in CKD"
    reference: "KDIGO 2020"

  # Literature effect sizes are declared here too, as `parameter` entries with
  # an `evidence_type` field — see §2.4. There is no separate `evidence:` block.
```

### 2.3 Distribution Expressions (Monte Carlo)

Parameter values may be expressed as distribution strings:

| Expression | Meaning |
|-----------|---------|
| `"normal(μ, σ)"` | Normal distribution |
| `"uniform(a, b)"` | Uniform distribution |
| `"lognormal(μ, σ)"` | Log-normal distribution |
| `"beta(α, β)"` | Beta distribution (for fractions) |
| A numeric literal | Deterministic value |

In Monte Carlo mode, each run samples all distribution-valued parameters exactly once (per-run, not per-step), representing inter-individual variability.

**Reproducibility**: A Monte Carlo session can be made reproducible by declaring a fixed seed in `optimizer.mc.seed`. When `seed` is omitted or `null`, the engine generates a random seed each session (suitable for exploration). When a fixed integer seed is declared, every run of the model produces identical trajectories, enabling result sharing and citation. The engine must return the actual seed used (`session_seed`) in its API response so users can record it.

---

## 2.4 Evidence: Literature Effect Sizes via `evidence_type`

Literature-derived effect sizes (RR, OR, HR, Cohen's d, incidence rates, regression coefficients, PK constants) are declared as ordinary `variables:` entries with `type: parameter` plus an additional `evidence_type` field. An OR=1.65 cannot enter an equation directly, but the Loader-computed effective coefficient can — `evidence_type` tells the Loader which conversion to apply. There is no separate top-level `evidence:` block; evidence is a variable like any other, just one whose raw literature value needs converting before equations can use it.

The Loader converts `value` in place at load time (same variable name, no `_effective` suffix) and additionally records two read-only provenance fields for inspection/audit: `evidence_type` (the original effect-size type) and `evidence_raw_value` (the literature value before conversion).

```yaml
variables:
  # Relative risk — used as multiplier directly, no conversion needed
  smoking_lung_cancer_rr:
    type: parameter
    evidence_type: rr
    value: 14.0
    reference: "Doll & Hill (1950) BMJ"

  # Odds ratio — requires baseline_prevalence for conversion
  obesity_diabetes_or:
    type: parameter
    evidence_type: or
    value: 1.65
    baseline_prevalence: 0.23
    reference: "..."

  # Hazard ratio — references a co-declared ir variable as baseline
  chemo_mortality_hr:
    type: parameter
    evidence_type: hr
    value: 0.82
    baseline_ref: chemotherapy_baseline_ir
    reference: "..."

  # Cohen's d — requires population_sd
  exercise_fev1_effect:
    type: parameter
    evidence_type: cohens_d
    value: 0.68
    population_sd: 0.5
    unit: L
    reference: "..."

  # Incidence rate
  annual_diabetes_ir:
    type: parameter
    evidence_type: ir
    value: 0.05
    unit: prob/year

  # Absolute risk difference
  statin_cvd_ard:
    type: parameter
    evidence_type: ard
    value: 0.012
    unit: prob/year

  # Regression coefficient
  age_bp_beta:
    type: parameter
    evidence_type: beta
    value: 0.45
    unit: mmHg/year

  # PK/PD parameter (directly measured)
  aspirin_ke:
    type: parameter
    evidence_type: pk
    value: 0.198
    unit: 1/hour
```

Declaring `evidence_type` on an entry whose `type` is not `parameter` is a load error.

### Loader Conversion Rules

| `evidence_type` | Conversion equation                          | Required auxiliary fields |
|-----------------|-----------------------------------------------|--------------------------|
| `rr`            | `effective = value`                         | —                        |
| `or`            | `effective = OR / ((1−p₀) + p₀×OR)`        | `baseline_prevalence`    |
| `hr`            | `effective = baseline_value × HR`           | `baseline_ref`           |
| `ard`           | `effective = value`                         | —                        |
| `cohens_d`      | `effective = d × population_sd`             | `population_sd`          |
| `ir`            | `effective = value`                         | —                        |
| `beta`          | `effective = value`                         | —                        |
| `pk`            | `effective = value`                         | —                        |

`baseline_ref` must point to another `variables:` entry in the same file declaring `evidence_type: ir` or `evidence_type: ard`.

### Optional automatic wiring: `applies_to`

For the five types whose wiring into a state variable's dynamics has exactly one unambiguous form (`ir`/`ard`/`hr`/`rr`/`or` — "the converted coefficient is a rate, accumulate it into a target state"), declaring `applies_to: <state_variable_name>` (plus `step_unit`, and `rate_unit` on the `ir`/`ard` side of any baseline) makes the Loader auto-generate the corresponding dynamics instead of the modeler hand-writing them. `cohens_d`, `beta`, and `pk` never support `applies_to` — their wiring is itself a modeling choice (functional form, regression structure, compartment structure) that the engine cannot assume; these three always require hand-written `dynamics`. Declaring `applies_to` on an entry without `evidence_type`, or on a `cohens_d`/`beta`/`pk` entry, is a load error, as is two evidence entries declaring the same `applies_to` target (how multiple risk factors combine — multiplicative vs. additive — is a modeling decision the engine does not make for you).

Evidence-derived parameters never participate in any optimization loop.

---

## 3. Equations

The `equations:` block defines dynamic equations and instantaneous algebraic relationships.

### 3.1 Equation Structure

```yaml
equations:
  gfr_decline:
    description: "GFR declines daily at rate modulated by protein intake"
    reference: "KDIGO 2020"
    condition: "gfr > 5"                # Equation only applies when condition is true
    priority: 1                         # Execution order within the same step (higher = first)
    dynamics:                           # State derivatives — multiplied by step automatically
      gfr: >
        gfr - gfr * gfr_decline_rate
        * (1 if protein_intake > 0.8 else protein_restriction_rr)
        * step

  muscle_protein_balance:
    description: "Net muscle protein synthesis minus catabolism"
    dynamics:
      muscle_mass: >
        muscle_mass
        + (protein_synthesis_rate * protein_intake - muscle_catabolism_rate)
        * (1 - ckd_catabolism_multiplier * max(0, (60 - gfr) / 60))
        * step

  ckd_stage_update:
    description: "Instantaneous CKD stage classification"
    dynamics:
      ckd_stage: >
        1 if gfr >= 90
        else 2 if gfr >= 60
        else 3 if gfr >= 30
        else 4 if gfr >= 15
        else 5
```

### 3.2 Equation Expression Language

Equation expressions are Python-compatible arithmetic strings. Available symbols:

| Symbol | Meaning |
|--------|---------|
| Any declared variable name | Current value of that variable |
| `step` | Current step duration in the declared `step_size.unit` |
| `t` or `time` | Elapsed time in declared unit |
| `SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK` | Absolute conversion constants |
| `sin`, `cos`, `exp`, `log`, `sqrt`, `abs`, `max`, `min` | Standard math functions |
| `if ... else ...` | Ternary conditional |

**Multiplier rule**: In `dynamics:` blocks, equations that represent rates must explicitly multiply by `step` to convert to a step-sized change. Algebraic assignments (e.g. `ckd_stage`, `performance`) do not use `step`.

### 3.3 Execution Order

Within a single time step, equations are processed in **one pass**, sorted by `priority`
in **descending** order (higher numeric `priority` runs first; default `0`).

For each equation, in order:
1. Evaluate `condition` (default `true`). If false, skip.
2. Evaluate `dynamics:` entries and write the results back to the model variables
   immediately, clamped to `bounds`.

**Sequential (Gauss-Seidel) update semantics**: writes happen immediately, so a equation
executed later in the pass (lower `priority`) reads the values already written by equations
executed earlier in the same step (higher `priority`) — not the values from the previous
step. If equation B depends on equation A's result for the *same* step, give A a higher
`priority` than B.

---

## 4. Imports

The `imports:` block enables model composition. Imports are resolved recursively.

```yaml
imports:
  - components/medical/renal/ckd_gfr_dynamics
  - components/medical/musculoskeletal/protein_muscle_synthesis
  - components/medical/metabolic/dietary_protein_kinetics
```

### 4.1 Import Resolution Rules

1. Import paths are relative to the model library root, without `.yaml` extension.
2. If a path names a directory, the engine loads `{directory}/{directory_name}.yaml` as the entry point.
3. Imports are resolved depth-first; circular imports are an error.
4. On variable/equation namespace conflicts, **the importing file wins** (root file overrides).
5. After full merge, all equation references must resolve to declared variables; unresolved references are a validation error.

### 4.2 Step Size in Multi-Model Scenarios

Each component may declare its own native `step_size`. When a Scenario imports components with different step sizes, LM-compatible engines must:
- Simulate each component at its native step size, or
- Convert all components to the Scenario's declared step size by rescaling rate coefficients.

The `step` symbol in equation expressions always equals the component's native step in its declared unit, ensuring equation coefficients remain stable regardless of the computational step chosen by the engine.

---

## 5. Simulation Block

The `simulation:` block makes an LM file a runnable Scenario. Time-varying inputs are declared inside one or more named `plans`; there is no bare top-level schedule list — every Scenario has at least one plan, even if only one.

```yaml
simulation:
  start_date: "2026-01-01"             # ISO 8601; step size declared in metadata.step_size
  end_date:   "2026-12-31"

  output_variables:                    # Optional: select specific variables by name
    - gfr
    - muscle_mass
    - ckd_stage

  output_types: [input, state]         # Optional: select all variables of listed types

  plans:                                # Required: at least one named plan
    - id: "kidney_first"
      label: "Kidney Protection Priority"
      regimens:                         # Time-varying input drivers for this plan
        - variable: protein_intake
          time_start: "08:00"           # HH:MM, 24-hour; see §5.1 for window-width defaults
          value: 0.8                    # total quantity delivered within the window
          days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]   # omit = every day
          date_range: ["2026-01-01", "2026-03-31"]     # omit = entire simulation period
          label: "Restriction phase"
        - variable: protein_intake
          time_start: "08:00"
          value: 1.0
          date_range: ["2026-04-01", "2026-12-31"]
          label: "Maintenance phase"
    - id: "muscle_first"
      label: "Muscle Preservation Priority"
      regimens:
        - variable: protein_intake
          time_start: "08:00"
          value: 1.2
          label: "High protein"
```

### 5.1 Regimen Entry Format

Each entry in a plan's `regimens` list declares a `[time_start, time_end)` window during which the total `value` is delivered, spread evenly across the steps that fall inside it (this is the *sustained* execution model — a window narrowed to a single step is numerically identical to what earlier tooling called a "pulse"). Outside the window the variable's contribution is 0. Fields:

| Field | Required | Description |
|-------|----------|-------------|
| `variable` | Yes | Must be a declared `type: input` variable |
| `time_start` | No | `HH:MM` 24-hour window start; see window-width defaults below |
| `time_end` | No | `HH:MM` 24-hour window end; see window-width defaults below |
| `value` | Yes | Total quantity delivered within the window (not a per-step rate); independent of step size or window width |
| `days` | No | Three-letter day list (`Mon`–`Sun`); omit = every day |
| `date_range` | No | `["YYYY-MM-DD", "YYYY-MM-DD"]`; omit = entire simulation period |
| `label` | No | Human-readable label for GUI display |
| `delivery` | No | `total` (default) or `level`, see below |

**Window-width defaults**: `time_start`/`time_end` are both optional, and omitting them is a deliberate choice, not missing configuration — engines resolve the window as follows:

| Written | Resulting window | Typical use |
|---|---|---|
| Neither given | Full day `["00:00", "24:00"]` | Day-rate inputs (daily caloric deficit, daily average intake) that have no meaningful "moment it happens" |
| Only `time_start` given | `time_end = time_start` (single-step window) | Discrete events (a meal, a dose) — numerically what earlier tooling called a "pulse" |
| Both given | Explicit interval | Sustained intensity/protection spanning multiple steps in a sub-day-step-size model |

**`delivery`: total vs. level (ADR 0132)**: with the default `delivery: total`, or `delivery` omitted, `value` is the window's total quantity, spread evenly across every step inside the resolved window, which suits a variable that downstream equations accumulate, such as a training load or a meal's calories. With `delivery: level`, `value` is instead delivered unchanged to every step inside the window rather than divided, which suits a variable whose value is a current state or setting that downstream equations read as an instantaneous quantity rather than sum, such as a sleep duration, a bedtime, or a care intensity. Both modes share the same window-width resolution above; `delivery` only changes how `value` is spread within the resolved window, not how the window itself is determined.

Multiple entries for the same variable with non-overlapping `date_range` represent the K×4 Regimen segments (see §7); the recommended way to express a multi-phase regimen is one `date_range`-free baseline entry (in effect for the whole simulation) plus one or more `date_range`-scoped entries carrying the *delta* relative to baseline — this way a missing or mistyped `date_range` on a delta entry only shifts a few days at the margin, rather than silently stacking a full duplicate target value on top of another phase's.

### 5.2 Plans

Each entry in `simulation.plans` is an independently runnable named scenario with its own `regimens` list. When a model is loaded, engines present all plans for parallel simulation. Plans do not override each other; each runs as an independent session.

### 5.3 Monte Carlo (simulation.mc)

`simulation.mc` controls the number of Monte Carlo runs for GUI visualization. It is **completely independent** from `optimizer.mc`; neither block inherits from the other.

```yaml
simulation:
  mc:
    runs: 30    # Number of visualization trajectories; omit or 1 = deterministic (single run)
    seed: 19    # Optional; fixed integer = reproducible, omit = new random seed each session
```

When `runs > 1`, the engine draws all distribution-valued `parameter` variables independently for each run. The engine must expose the actual seed used (`session_seed`) in its API response so results can be reproduced. The GUI renders N semi-transparent thin lines plus one mean line.

`algorithm.seed` (in `optimizer.algorithm`) controls only the NSGA-II population initialization; it has no effect on MC sampling.

---

## 6. Optimizer Block

The `optimizer:` block is a **top-level key** (not nested under `simulation:`). It defines a multi-objective optimization problem whose decision variables are `input` variable schedules. When present, a conforming engine searches for Pareto-optimal K×4 Regimens (see §7). After optimization completes, results are written back into `optimizer.results`.

### Independence Principle

`simulation:` and `optimizer:` are independent scenario descriptions that may be converted between each other by an interactive tool. A file may contain one, both, or neither. The optimizer block defines its own evaluation context; fields absent from the optimizer block fall back to their simulation-block equivalents.

| Concern | simulation | optimizer |
|---------|-----------|-----------|
| Time range | `simulation.start_date` / `end_date` | `optimizer.start_date` / `end_date` (optional override) |
| Step size | `metadata.step_size` | `optimizer.step_size` (optional override) |
| Fixed inputs + Decision variables | `simulation.plans[*].regimens` (visualization) | `optimizer.startpoint.regimens` (unified list, independent evaluation) |

`optimizer.startpoint.regimens` is a unified list of both fixed background inputs (entries with no `optimize:` block) and decision variables (entries with an `optimize:` block); `startpoint` describes the initial protocol the optimizer searches from. The optimizer never reads `simulation.plans[*].regimens` — the two paths are fully independent, and `optimizer.startpoint.regimens` must be declared explicitly (no implicit fallback between them).

```yaml
optimizer:
  method: nsga2                        # nsga2 | l-bfgs-b | nelder-mead

  # Evaluation time window (optional; defaults to simulation.start_date/end_date + metadata.step_size)
  start_date: "YYYY-MM-DD"            # Optional; overrides simulation.start_date for optimizer evaluation
  end_date:   "YYYY-MM-DD"            # Optional; overrides simulation.end_date for optimizer evaluation
  step_size:                           # Optional; overrides metadata.step_size for optimizer evaluation
    value: 1
    unit: day                          # minute | hour | day

  objectives:
    - variable: muscle_mass
      metric: final                    # final | max | min | mean
      direction: maximize
    - variable: gfr
      metric: final
      direction: maximize

  constraints:                         # Optional
    - variable: gfr
      condition: ">= 15"               # supports <=, >=, <, >, ==
      type: hard                       # hard | soft
    - variable: protein_intake
      condition: "<= 1.2"
      type: hard

  algorithm:                           # Optional; defaults pop=50, gen=80, seed=42
    population_size: 100
    n_generations: 200
    seed: 19                           # NSGA-II genetic-algorithm seed, unrelated to MC

  # Monte Carlo settings (optional; controls inner-loop variability sampling during optimization)
  # Completely independent from simulation.mc — do NOT inherit from each other.
  mc:
    runs: 5                  # Inner MC runs per candidate evaluation; omit or 1 = deterministic
    seed: 19                 # Optional; fixed integer = reproducible, omit = random each session

  startpoint:
    regimens:                          # Decision variables + fixed background inputs, unified list
      - variable: background_drug      # Fixed background quantity (no optimize: block)
        time_start: "08:00"
        value: 500
        days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
      - variable: protein_intake       # T1: value search (decision variable)
        time_start: "08:00"
        days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
        label: "Protein dose"
        optimize:
          value: [0.3, 1.5]            # [lo, hi] search range

  results:                             # Written back by the GUI after optimization completes; no manual editing needed
    generated_at: "2026-05-10"
    method: nsga2
    n_solutions: 12
    elapsed_seconds: 87.3
    pareto_front:                      # All non-dominated solutions (flow-style, one per line)
      - {x: [0.60], f: [62.1, 48.3]}
      - {x: [0.80], f: [64.8, 46.2]}
      - {x: [1.00], f: [66.9, 43.8]}
    recommended:                        # Modeler-selected representative point (not the unique optimum)
      x: [0.80]                        # Decision-variable values, in optimizer.startpoint.regimens decision-entry order
      f: [64.8, 46.2]                  # Objective values, in objectives order
```

Human-readable regimens/objective dictionaries are not stored alongside `x`/`f` — both the human-readable display and "send to Sim" reconstruct them on demand by decoding `x`/`f` against `optimizer.startpoint.regimens` and `objectives` (avoiding two formats that can drift out of sync).

### 6.1 Evaluation Time Window

The `optimizer` block may declare an independent evaluation time window (`start_date` / `end_date` / `step_size`). This window governs the simulation that the engine runs internally on each fitness evaluation. When absent, the engine inherits values from `simulation.start_date`/`end_date` and `metadata.step_size`.

**Use cases:**
- **Reproducibility**: when `optimizer.results` is published, readers can reproduce the Pareto front by rerunning with the declared evaluation window.
- **Accelerated search**: a shorter evaluation window (e.g., 2 years instead of 5) reduces per-evaluation cost while preserving search quality, provided the objective variable reaches a stable value within the window.
- **Decoupled visualization**: `simulation.start_date`/`end_date` controls the GUI display; `optimizer.start_date`/`end_date` controls what the optimizer actually evaluates — these may differ.

A conforming engine must apply the following priority when resolving evaluation time:
1. Values passed by the interactive session (e.g., GUI toolbar) as runtime overrides — highest priority.
2. `optimizer.start_date` / `end_date` / `step_size` — declared in the LM file.
3. `simulation.start_date` / `end_date` and `metadata.step_size` — inherited defaults.

### 6.2 optimizer.results Design Principles

- **Results travel with the model**: `optimizer.results` is serialized into the same YAML file as the model. Publishing the model = publishing the Pareto front.
- **`recommended` is not `best`**: Multi-objective optimization has no unique optimum. `recommended` is a researcher-selected representative point from the Pareto front, chosen to illustrate a specific tradeoff. Users should inspect the full `pareto_front`. The field is named `recommended` rather than `reference` to avoid colliding with the unrelated `reference` field used elsewhere for literature citations (on `variables.<name>` / `equations.<name>`).
- **Warm-start**: Engines may initialize subsequent runs from `pareto_front` vectors to continue improving the front.
- **Overwrite on save**: `results` is wholly replaced each time the researcher saves; no append semantics.

---

## 7. K×4 Regimen Specification

**K×4 Regimen** is LM format's canonical input formalism for behavioral intervention optimization. It is a first-class part of the LM format specification, not an engine feature.

### 7.1 Motivation

Clinical interventions are not arbitrary continuous control signals — they are schedules that a real person must execute. A medication regimen is "take 2 tablets at 8am and 8pm." An exercise plan is "30 minutes of walking on Monday, Wednesday, Friday." A dietary plan is "reduce protein to 0.6 g/kg/day for the first 4 weeks."

Standard continuous optimal control outputs (a control signal u(t) for every time step) are:
- Computationally expensive (search space scales as T × N, where T is time steps and N is variables)
- Clinically uninterpretable and unexecutable by patients

K×4 Regimen addresses this by formalizing interventions as a finite set of **calendar-semantic segments**.

### 7.2 Formal Definition

A **K×4 Regimen** for a single input variable is defined as:

```
R = {(t_k, d_k, p_k, v_k) : k = 1, ..., K}
```

Where:
- `K` is the number of schedule segments (the design choice)
- `t_k` ∈ ℝ: start time of segment k (in declared unit)
- `d_k` ∈ ℝ₊: duration of segment k
- `p_k` ∈ {daily, weekly, ...}: periodicity (how often within the segment)
- `v_k` ∈ ℝ: dose or value for each occurrence in segment k

The search space of a K×4 Regimen over K segments is **4K dimensions**, independent of simulation time T. For typical clinical interventions (K = 2–6), this reduces the search space by a factor of T/4K relative to step-by-step control, often 30× or more.

### 7.3 Calendar Semantics

K×4 Regimen outputs are directly human-readable and exportable:
- Each segment maps to a block of calendar entries (iCal format)
- `t_k` and `d_k` map to DTSTART and DURATION
- `p_k` maps to RRULE (DAILY, WEEKLY, etc.)
- `v_k` maps to the event description (dose, duration, intensity)

This makes K×4 Regimen the **only optimization output format** in LM format that can be directly given to a patient as an executable plan.

### 7.4 YAML Representation

A K×4 Regimen is expressed in a plan's `regimens` list (§5.1) as multiple entries for the same variable, each with a distinct `date_range` (one entry per segment):

```yaml
simulation:
  plans:
    - id: default
      regimens:
        - variable: protein_intake      # segment 1: t₁=day 0, d₁=60 days, p₁=daily, v₁=0.6
          time_start: "08:00"
          value: 0.6
          date_range: ["2026-01-01", "2026-03-01"]
          label: "Restriction phase"
        - variable: protein_intake      # segment 2: t₂=day 60, d₂=90 days, p₂=daily, v₂=0.8
          time_start: "08:00"
          value: 0.8
          date_range: ["2026-03-02", "2026-06-29"]
          label: "Relaxation phase"
        - variable: protein_intake      # segment 3: t₃=day 150, d₃=215 days, p₃=daily, v₃=1.0
          time_start: "08:00"
          value: 1.0
          date_range: ["2026-06-30", "2026-12-31"]
          label: "Maintenance phase"
```

A solved K×4 Regimen from optimization is serialized as an `x` vector in `optimizer.results.recommended` (see §6), decoded against the decision entries in `optimizer.startpoint.regimens` in the same order — engines reconstruct the human-readable per-segment values from `x` on demand rather than storing a separate decoded dictionary (see §6, "optimizer.results Design Principles").

The underlying `time_start`/`date_range` YAML is human-readable and can be exported as iCal: each segment maps to one VEVENT with DTSTART, DURATION, RRULE (daily), and DESCRIPTION (dose value).

---

## 8. Model Library Standards

An LM file is eligible for submission to the **LM Open Model Library** if it meets all of the following:

### 8.1 Required Fields

- `metadata.name`, `metadata.version`, `metadata.description`
- `metadata.authors` (at least one author with name)
- `metadata.references` (at least one entry for each literature-derived parameter, or a `reference` on the individual `variables`/`equations` entry it supports)
- `metadata.lm_format_version: "1.0"`

### 8.2 Validation

A Library-eligible LM file must pass validation by an LM-compatible engine:
- All variable references in equations resolve to declared variables
- All import paths resolve
- Initial values are within declared bounds
- No circular imports

### 8.3 Quality Status: `metadata.todo`

Whether an LM file is publication-ready is tracked entirely by a structured `metadata.todo` field, independent of the file name:

```yaml
metadata:
  todo:
    - type: nosim | noopt | noref | quality | other
      issue: "One-sentence description of the problem"
      evidence: "Diagnostic basis: specific numbers/behavior/how to reproduce, so the next pass doesn't have to re-diagnose"
      next: "Suggested next step, or the options left for a human to decide — this field does not decide for them"
```

- `type` meanings: `nosim` = `--sim` cannot run; `noopt` = `--sim` passes but the optimizer fails; `noref` = missing literature sourcing (a `TODO:SOURCE` marker is present); `quality` = both `--sim`/`--opt` succeed but the result is suspect (e.g. a degenerate Pareto front, an empty feasible region); `other` = anything else.
- **No `metadata.todo` (or an empty list) means confirmed passing and publication-ready**: `--sim` succeeds, `--opt` succeeds (or is skipped when no `optimizer:` block is present), every parameter has a literature source, and results show no red flags.
- Once every `todo` item is resolved, remove the field entirely — the file returns to a "clean" state. The file name is never part of this signal (a prior convention encoded status via filename suffixes such as `_HOLD`; renaming broke other files' `imports:` paths and has been removed).

An optional YAML field `metadata.reviewed: true` may be set by the author to indicate that mechanisms and parameter magnitudes have been manually verified. This is not a publication gate.

### 8.4 Contribution License Agreement

By contributing an LM file to the LM Open Model Library, contributors confirm:
1. They have the right to submit the contribution.
2. They agree to license their model contribution under the terms stated in the receiving repository's `LICENSE` file, currently CC BY 4.0 for YAML model content.
3. The contribution does not reproduce verbatim text from copyrighted materials.

---

## 9. Versioning

### 9.1 LM format Version History

| Version | Date | Status | Notes |
|---------|------|--------|-------|
| 1.0 | 2026-05-10 | Draft | Initial release |
| 1.0 | 2026-05-19 | Draft | Updated to align with Reference Engine: `evidence:` as separate top-level block with 8 sub-types; `simulation.schedules` updated to flat-list pulse format; `simulation.plans` added; `optimizer:` moved to top-level; `optimizer.results.reference` (replaces `best`); `metadata.description` structured form; `simulation.step_size` removed (only in `metadata.step_size`) |
| 1.0 | 2026-05-22 | Draft | Added `optimizer.start_date` / `end_date` / `step_size` (optional evaluation time window, §6.1); optimizer schedule tiers T2/T3/T4 introduced |
| 1.0 | 2026-05-23 | Draft | Formalized Sim / Opt independence principle (§6 intro) |
| 1.0 | 2026-05-28 | Draft | **Breaking**: `optimizer.inputs` deprecated → unified `optimizer.schedules` (ADR 0088); T2 field `optimize.time: ["HH","HH"]`; T3 field `optimize.days_pool + days_n` (backend enumerates combinations, replaces explicit `days_options`); T4 field `optimize.date_range: [[lo,hi],[lo,hi]]` (two mandatory windows); all legacy fields (`time_window`, `opt_step`, `days_options`, `date_start_window`, boolean flags) removed from engine and all YAMLs |
| 1.0 | 2026-05-24 | Draft | Added `optimizer.mc.seed` (optional integer; fixed = reproducible MC, omit = random each session); reproducibility note added to §2.3; Reference Engine returns `session_seed` in API response |
| 1.0 | 2026-06-06 | Draft | **ADR 0096**: Added §8.3 File Naming Quality Markers — three-suffix convention (`_nosim`, `_noopt`, `_noref`); deprecated `_mw` and `_TODO`; added `metadata.reviewed` optional field. |
| 1.0 | 2026-06-05 | Draft | **ADR 0092**: `type: input` variable `unit` field must be a bare event-quantity unit (e.g. `mg`, `g/kg`, `kcal`, `MET-h`); rate units (`mg/day`, `kcal/day`, `g/kg/day`) are prohibited — they belong in `description` or `reference`; continuous rate processes must be modeled as `parameter` with step-multiplied equations. **ADR 0045 update**: `simulation.mc` and `optimizer.mc` are now separate, fully independent blocks; `optimizer.mc.enabled` and `optimizer.mc.sim_runs` deprecated → use `runs:` in each block; `algorithm.seed` controls only NSGA-II and does not fall back as mc.seed. §5.3 added for `simulation.mc`; §6 mc block updated. |
| 1.0 | 2026-07-30 | Draft | Removed former §8 Game Conversion Block (`game:` top-level key, "Story" term) — never implemented in Reference Engine, GUI, or any model YAML; the feature this section described does not exist. Sections renumbered §9-§13 → §8-§12 accordingly. Will be reintroduced as a new versioned addition if/when the LM Game conversion mechanism is actually built. |
| 1.0 | 2026-07-30 | Draft | §9.3 Planned Extensions: removed "multi-individual simulation (household, cohort scenarios)" — LM format's individual-level scope (Scope section) is explicitly per-individual batch execution (independent MC samples, no inter-individual interaction), not multi-agent/cohort simulation; this was never a planned direction. |
| 1.0 | 2026-07-30 | Draft | Added Inclusion Test to the Scope section — a four-question operational check (`variables`/`evidence`, `equations`, `simulation`, `optimizer`) for whether a candidate topic falls within LM format's scope. The discipline-by-discipline coverage inventory this test produces is tracked in the LM Reference Library's `docs/model.md`, not versioned with this specification. |
| 1.0 | 2026-07-30 | Draft | Reconciled the spec with the Reference Engine's current implementation, which had drifted from several sections: **ADR 0137** — `evidence:` is no longer an independent top-level block; literature effect sizes are declared as `variables:` entries with `type: parameter` + `evidence_type` (§2.4 rewritten, `applies_to` auto-wiring documented). **ADR 0109/0127** — `simulation.schedules`/`time:` replaced by `simulation.plans[*].regimens`/`time_start`+`time_end` (sustained execution model with window-width defaults; §5, §5.1, §7.4 rewritten); bare top-level `simulation.schedules` no longer supported. **ADR 0088/0109** — `optimizer.inputs`/`optimizer.schedules` replaced by `optimizer.startpoint.regimens`; `objectives` now `{variable, metric, direction}` (not `{maximize, at_time}`); `constraints` now `{variable, condition, type}` (not `{variable, operator, value}`); `optimizer.results.reference` renamed `optimizer.results.recommended`, storing only `{x, f}` vectors (decoded regimen/objective dictionaries removed 2026-06-21) (§6 rewritten). **ADR 0120** — §8.3 rewritten from the abolished `_nosim`/`_noopt`/`_noref` filename-suffix convention to the current `metadata.todo` structured field. |
| 1.0 | 2026-08-06 | Draft | Added `delivery: total \| level` to §5.1 (ADR 0132), a regimen field distinguishing accumulated quantities from instantaneous state/setting readings, implemented since 2026-07-15 but missing from this spec until now. The LM Reference Library's practitioner documentation, previously the single file `docs/model.md`, has been split into a `docs/authoring/` directory; pointers elsewhere in this spec to `docs/model.md`, including the Scope section and the 2026-07-30 row above, now resolve via `docs/authoring/README.md`. |
| 1.0 | 2026-08-06 | Draft | Removed `metadata.citation`, `metadata.sources`/`source_id`, `metadata.license`/`license_url`, `metadata.status`, and `authors[].orcid`/`affiliation`/`role` from §1.2, and the matching `source_id` examples from §2.2/§3.1, none of these were ever read by the Reference Engine loader or used in any model YAML in the library, the same situation as the 2026-07-30 removal of the Game Conversion Block. Literature sourcing is declared via `metadata.references` and the per-`variables`/`equations` `reference` field, both already implemented; §1.2's `authors` example now matches the `name`/`email` shape actually in use; the `tags` example changed from a list of single-key mappings, never used in practice, to the flat string list every model YAML actually uses. §8.1 Required Fields and §8.4 Contribution License Agreement updated to match. These fields may return as a new versioned addition if a Model Library submission mechanism that needs them is actually built. |
| 1.0 | 2026-08-08 | Draft | **Breaking**: top-level `formulas:` block renamed to `equations:` (ADR 0144), matching the terminology already used throughout this spec's own prose (`differential equation`, `dynamic equations`); the `variables`/`formulas`/`simulation`/`optimizer` shorthand introduced by the 2026-07-30 Inclusion Test row is now named V.E.S.O. — Variables, Equations, Simulation, Optimization. All model YAML files, the Reference Engine, and the GUI updated in the same pass. |

### 9.2 Versioning Policy

LM format uses semantic versioning:
- **Patch** (1.0.x): Clarifications, editorial fixes, no schema changes
- **Minor** (1.x.0): Backward-compatible additions (new optional fields)
- **Major** (x.0.0): Breaking schema changes

LM files declare `metadata.lm_format_version` to indicate which version of the specification they target. LM-compatible engines should accept files targeting older LM format versions.

### 9.3 Planned Extensions (LM format 1.1+)

- Validation metadata (uncertainty ranges, sensitivity indices)
- Model composition constraints (required/prohibited imports)
- Probabilistic event modeling (stochastic state transitions) — not to be confused with the already-implemented Monte Carlo parameter sampling (§2.3/§5.3/§6): MC draws each distribution-valued `parameter` once per run to represent inter-individual variability, while this planned extension is about a state variable making a stochastic transition *during* a run (e.g. a Markov-style jump), which no equation expression can currently do (§3.2 lists no random-draw function)

Features already implemented in Reference Engine (backported into LM format 1.0): `evidence_type`-based effect-size declarations (§2.4), `simulation.plans[*].regimens` (§5), `optimizer.results` (§6).

---

## 10. Reference Implementation

The **LM Reference Engine** is the first software implementation of this specification.

```
LM Reference Engine
Initial author: Fan Shen
Institutional affiliation: Sun Yat-sen University, School of Systems Science and Engineering
Repository: [GITHUB_URL]
License: PolyForm Noncommercial 1.0.0 (engine code); CC BY 4.0 (documentation, YAML schemas)
```

The Reference Engine is **one possible implementation** of LM format. Other engines — commercial or open-source — may implement LM format compatibility independently. A conforming engine must:

1. Load and parse LM format 1.0 YAML files
2. Resolve imports recursively
3. Validate all variable references and bounds
4. Execute equations in the declared priority order with correct step-scaling
5. Support K×4 Regimen as an optimization input formalism
6. Serialize solved Regimens in the format defined in §6 and §7.4

---

## 11. Citation

If you use LM format in a publication, please cite:

```bibtex
@techreport{shen2026lmformat,
  title     = {{Life Matters Format (LM format) 1.0: An Open YAML Specification
                for Executable Health and Social Dynamics Models}},
  author    = {Shen, Fan},
  year      = {2026},
  institution = {Sun Yat-sen University, School of Systems Science and Engineering},
  note      = {Available at: [GITHUB_URL]. DOI: [ZENODO_DOI]}
}
```

If you use the LM Reference Engine, additionally cite:

```bibtex
@article{shen2026lmengine,
  title   = {{Life Matters: A YAML Model Format and Reference Engine for
              Health and Social Dynamics Simulation}},
  author  = {Shen, Fan},
  journal = {[TARGET_JOURNAL]},
  year    = {2026},
  note    = {Under review}
}
```

---

## 12. Governance

LM format 1.0 is authored and maintained by Fan Shen. The specification is intended to evolve as a **shared research commons** — meaning:

- The format itself is public and open.
- Anyone may implement an LM-compatible engine.
- Contributions to the format specification may be proposed via the public repository.
- Commercial products may implement or support the LM Format; the LM name should primarily identify the open specification, model library, and scholarly community.

> LM is intended to function as a shared research commons rather than an exclusive commercial brand. Commercial products may implement or support the LM Format, but the LM name should primarily identify the open specification, model library, and scholarly community.

---

*Life Matters Format (LM format) 1.0 — Initial draft, Fan Shen, 2026-05-10*  
*Sun Yat-sen University, School of Systems Science and Engineering*
