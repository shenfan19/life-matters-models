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

---

## Terms and Definitions

| Term | Definition |
|------|-----------|
| **LM format** | Life Matters format — this specification |
| **LM file** | A YAML file conforming to this specification |
| **Component** | An LM file defining reusable physiological or behavioral sub-models |
| **Scenario** | An LM file that includes a `simulation:` block and is directly runnable |
| **Story** | An LM file that includes a `game:` block for card-game conversion |
| **LM-compatible engine** | Any software that can load, validate, and execute LM files according to this specification |
| **LM Reference Engine** | The first LM-compatible engine, authored by Fan Shen at Sun Yat-sen University |
| **K×4 Regimen** | A calendar-semantic input formalism defined in this specification (§6) |
| **Model Library** | A curated collection of LM files, open for contribution and citation |
| **step** | The canonical time step duration as declared in `metadata.step_size` |

---

## 1. File Format

LM files are YAML 1.2 documents. A conforming LM file must have a `metadata:` block and at least one of: `variables:`, `formulas:`, `imports:`. All keys are lowercase snake_case.

### 1.1 Top-Level Structure

```yaml
# Top-level keys in an LM file (all optional except metadata)
metadata:       # Required. Identification, versioning, citation, license.
imports:        # Optional. References to other LM files to merge.
variables:      # Optional. state, input, and parameter declarations.
evidence:       # Optional. Literature-derived effect sizes (RR, OR, HR, Cohen's d, etc.).
formulas:       # Optional. Dynamic equations and instantaneous calculations.
simulation:     # Optional. Makes this file a runnable Scenario.
optimizer:      # Optional. Multi-objective optimization configuration and results.
game:           # Optional. Defines game-conversion hints for a Story.
```

### 1.2 Metadata Block

```yaml
metadata:
  # --- Identity (required) ---
  name: "ckd_protein_muscle"           # Unique identifier, snake_case
  version: "1.0"                        # Semantic version of this model file

  # description accepts a plain string or a structured mapping.
  # Structured form: any subset of recommended keys (brief, need, problem, method,
  # simulation, optimization, result, conclusion, limitations) plus custom keys.
  # GUI displays all non-empty fields in YAML key order.
  description:
    brief: "CKD protein-muscle tradeoff model."
    problem: "Bidirectional conflict: low protein protects kidneys but accelerates muscle wasting."
    method: "Euler ODE, day-scale, coupled via LM format imports."
  # Plain string form is also valid: description: "CKD protein-muscle tradeoff model."

  # --- Classification (recommended) ---
  tags:
    - domain: medical
    - system: renal
    - system: musculoskeletal
    - disease: chronic_kidney_disease
    - scale: day
    - scale: week
  lm_format_version: "1.0"                    # Which LM format version this file targets

  # --- Time scale (required for Scenarios; recommended for Components) ---
  step_size:
    value: 1
    unit: day                           # second | minute | hour | day | week | month | year

  # --- Authorship (required for Model Library submission) ---
  authors:
    - name: "Fan Shen"
      orcid: "0000-XXXX-XXXX-XXXX"
      affiliation: "Sun Yat-sen University"
      role: "model_author"
    - name: "Example Collaborator"
      role: "parameter_contributor"

  # --- Citation (required for Model Library submission) ---
  citation:
    preferred: >
      Shen F. (2026). Life Matters Format (LM format): CKD protein-muscle
      tradeoff model v1.0. [Model Library entry]. DOI: 10.XXXX/lmml.ckd.v1
    doi: "10.XXXX/lmml.ckd.v1"          # Zenodo or equivalent DOI
    format_citation: >
      Shen F. (2026). Life Matters Format (LM format) 1.0. [Specification].
      Sun Yat-sen University. DOI: 10.XXXX/lmml-spec.v1

  # --- Sources (required if parameters come from literature) ---
  sources:
    - id: "kdigo2020"
      reference: "KDIGO 2020 Clinical Practice Guideline for Diabetes Management in CKD"
      doi: "10.1016/j.kint.2020.06.019"
      used_for: ["gfr_decline_rate", "protein_restriction_threshold"]
    - id: "bauer2013"
      reference: "Bauer J et al. (2013). Sarcopenia in CKD. NDT."
      doi: "10.1093/ndt/gft072"
      used_for: ["muscle_loss_rate_ckd"]

  # --- License (required) ---
  license: "CC BY 4.0"
  license_url: "https://creativecommons.org/licenses/by/4.0/"

  # --- Status ---
  status: "validated"                   # draft | provisional | validated | deprecated
  created: "2026-05-10"
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

Literature-derived effect sizes (RR, OR, HR, Cohen's d, incidence rates, regression coefficients, and PK constants) are declared in the `evidence:` top-level block, not under `variables:`. See §2.4.

### 2.2 Variable Declaration

```yaml
variables:
  gfr:
    type: state
    value: 45.0                         # Initial value
    unit: "mL/min/1.73m²"
    bounds: [0, 120]                    # [min, max] — hard physiological limits
    description: "Glomerular filtration rate"
    source_id: "kdigo2020"              # Reference from metadata.sources

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
    source_id: "kdigo2020"

  # Literature effect sizes go in the evidence: block, not here — see §2.4
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

## 2.4 Evidence Block

The `evidence:` block (top-level, not nested under `variables:`) declares literature-derived effect sizes that require unit conversion before use in formulas. An OR=1.65 cannot enter an equation directly, but the Loader-computed effective risk multiplier can.

LM-compatible engines must apply the following conversion rules at load time. Formulas reference the variable name; the engine automatically supplies the `_effective` value.

```yaml
evidence:
  # Relative risk — used as multiplier directly
  smoking_lung_cancer_rr:
    type: rr
    value: 14.0
    reference: "Doll & Hill (1950) BMJ"

  # Odds ratio — requires baseline_prevalence for conversion
  obesity_diabetes_or:
    type: or
    value: 1.65
    baseline_prevalence: 0.23
    reference: "..."

  # Hazard ratio — references a co-declared ir variable
  chemo_mortality_hr:
    type: hr
    value: 0.82
    baseline_ref: chemotherapy_baseline_ir
    reference: "..."

  # Cohen's d — requires population_sd
  exercise_fev1_effect:
    type: cohens_d
    value: 0.68
    population_sd: 0.5
    unit: L
    reference: "..."

  # Incidence rate (replaces deprecated probability_constant type)
  annual_diabetes_ir:
    type: ir
    value: 0.05
    unit: prob/year

  # Absolute risk difference
  statin_cvd_ard:
    type: ard
    value: 0.012
    unit: prob/year

  # Regression coefficient
  age_bp_beta:
    type: beta
    value: 0.45
    unit: mmHg/year

  # PK/PD parameter (directly measured)
  aspirin_ke:
    type: pk
    value: 0.198
    unit: 1/hour
```

### Loader Conversion Rules

| `type`      | Conversion formula                          | Required auxiliary fields |
|-------------|---------------------------------------------|--------------------------|
| `rr`        | `effective = value`                         | —                        |
| `or`        | `effective = OR / ((1−p₀) + p₀×OR)`        | `baseline_prevalence`    |
| `hr`        | `effective = baseline_ir × HR`              | `baseline_ref`           |
| `ard`       | `effective = value`                         | —                        |
| `cohens_d`  | `effective = d × population_sd`             | `population_sd`          |
| `ir`        | `effective = value`                         | —                        |
| `beta`      | `effective = value`                         | —                        |
| `pk`        | `effective = value`                         | —                        |

Evidence variables never participate in any optimization loop. `probability_constant` is a deprecated type alias for `type: ir`.

---

## 3. Formulas

The `formulas:` block defines dynamic equations and instantaneous algebraic relationships.

### 3.1 Formula Structure

```yaml
formulas:
  gfr_decline:
    description: "GFR declines daily at rate modulated by protein intake"
    source_id: "kdigo2020"
    condition: "gfr > 5"                # Formula only applies when condition is true
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
    formula:                            # Instantaneous — NOT multiplied by step
      ckd_stage: >
        1 if gfr >= 90
        else 2 if gfr >= 60
        else 3 if gfr >= 30
        else 4 if gfr >= 15
        else 5
```

### 3.2 Formula Expression Language

Formula expressions are Python-compatible arithmetic strings. Available symbols:

| Symbol | Meaning |
|--------|---------|
| Any declared variable name | Current value of that variable |
| `step` | Current step duration in the declared `step_size.unit` |
| `t` or `time` | Elapsed time in declared unit |
| `SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK` | Absolute conversion constants |
| `sin`, `cos`, `exp`, `log`, `sqrt`, `abs`, `max`, `min` | Standard math functions |
| `if ... else ...` | Ternary conditional |

**Multiplier rule**: In `dynamics:` blocks, formulas must explicitly multiply by `step` to convert a rate into a step-sized change. In `formula:` blocks (instantaneous), `step` is never multiplied.

### 3.3 Execution Order

Within a single time step, formulas are processed in **one pass**, sorted by `priority`
in **descending** order (higher numeric `priority` runs first; default `0`).

For each formula, in order:
1. Evaluate `condition` (default `true`). If false, skip the entire formula (both
   `dynamics:` and `formula:`).
2. Evaluate `dynamics:` entries and write the results back to the model variables
   immediately, clamped to `bounds`.
3. Evaluate dict-form `formula:` entries (`{var: expr}`) and write back, same as `dynamics:`.
4. Evaluate string-form `formula:` (instantaneous expression) and store the result
   under the formula's name (not written to a variable).

**Sequential (Gauss-Seidel) update semantics**: writes happen immediately, so a formula
executed later in the pass (lower `priority`) reads the values already written by formulas
executed earlier in the same step (higher `priority`) — not the values from the previous
step. If formula B depends on formula A's result for the *same* step, give A a higher
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
4. On variable/formula namespace conflicts, **the importing file wins** (root file overrides).
5. After full merge, all formula references must resolve to declared variables; unresolved references are a validation error.

### 4.2 Step Size in Multi-Model Scenarios

Each component may declare its own native `step_size`. When a Scenario imports components with different step sizes, LM-compatible engines must:
- Simulate each component at its native step size, or
- Convert all components to the Scenario's declared step size by rescaling rate coefficients.

The `step` symbol in formula expressions always equals the component's native step in its declared unit, ensuring formula coefficients remain stable regardless of the computational step chosen by the engine.

---

## 5. Simulation Block

The `simulation:` block makes an LM file a runnable Scenario.

```yaml
simulation:
  start_date: "2026-01-01"             # ISO 8601; step size declared in metadata.step_size
  end_date:   "2026-12-31"

  output_variables:                    # Optional: select specific variables by name
    - gfr
    - muscle_mass
    - ckd_stage

  output_types: [input, state]         # Optional: select all variables of listed types

  schedules:                           # Time-varying input drivers (flat list, pulse mode)
    - variable: protein_intake
      time: "08:00"                    # HH:MM, 24-hour
      value: 0.8
      days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]   # omit = every day
      date_range: "2026-01-01 ~ 2026-03-31"        # omit = entire simulation period
      label: "Restriction phase"
    - variable: protein_intake
      time: "08:00"
      value: 1.0
      date_range: "2026-04-01 ~ 2026-12-31"
      label: "Maintenance phase"

  plans:                               # Optional: pre-defined named plans for multi-scenario comparison
    - id: "kidney_first"
      label: "Kidney Protection Priority"
      schedules:
        - variable: protein_intake
          time: "08:00"
          value: 0.6
          label: "Low protein"
    - id: "muscle_first"
      label: "Muscle Preservation Priority"
      schedules:
        - variable: protein_intake
          time: "08:00"
          value: 1.2
          label: "High protein"
```

### 5.1 Schedule Format

Each entry in `schedules` is a pulse-mode event: the engine writes `value` at the specified `time` on matching days; all other steps receive 0. Fields:

| Field | Required | Description |
|-------|----------|-------------|
| `variable` | Yes | Must be a declared `type: input` variable |
| `time` | Yes | `HH:MM` 24-hour trigger time |
| `value` | Yes | Pulse value at trigger |
| `days` | No | Three-letter day list (`Mon`–`Sun`); omit = every day |
| `date_range` | No | `YYYY-MM-DD ~ YYYY-MM-DD`; omit = entire simulation period |
| `label` | No | Human-readable label for GUI display |

Multiple entries for the same variable with non-overlapping `date_range` represent the K×4 Regimen segments (see §7).

### 5.2 Plans

`simulation.plans` allows researchers to pre-define named comparison scenarios in the YAML file. Each plan has its own `schedules` list. When a model is loaded, engines present all plans for parallel simulation. Plans do not override each other; each runs as an independent session.

`simulation.schedules` (without `plans`) is the legacy single-plan format and remains supported.

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
| Fixed inputs + Decision variables | `simulation.schedules` | `optimizer.schedules` (unified list) |

`optimizer.schedules` is a unified list of both fixed background inputs (no `optimize:` block) and decision variables (with `optimize:` block). It replaces the former separate `optimizer.inputs` key (deprecated as of ADR 0088, 2026-05-28).

```yaml
optimizer:
  method: nsga2                        # nsga2 | lbfgsb | nelder_mead

  # Evaluation time window (optional; defaults to simulation.start_date/end_date + metadata.step_size)
  start_date: "YYYY-MM-DD"            # Optional; overrides simulation.start_date for optimizer evaluation
  end_date:   "YYYY-MM-DD"            # Optional; overrides simulation.end_date for optimizer evaluation
  step_size:                           # Optional; overrides metadata.step_size for optimizer evaluation
    value: 1
    unit: day                          # second | minute | hour | day

  objectives:
    - maximize: muscle_mass
      at_time: 365
    - maximize: gfr
      at_time: 365

  constraints:
    - variable: gfr
      operator: ">="
      value: 15
    - variable: protein_intake
      operator: "<="
      value: 1.2

  algorithm:
    pop_size: 100
    n_gen: 200

  # Monte Carlo settings (optional; controls inner-loop variability sampling during optimization)
  # Completely independent from simulation.mc — do NOT inherit from each other.
  mc:
    runs: 5                  # Inner MC runs per candidate evaluation; omit or 1 = deterministic
    seed: 19                 # Optional; fixed integer = reproducible, omit = random each session

  # Fixed background inputs during evaluation (optional; same format as simulation.schedules)
  schedules:
    - variable: background_drug
      time: "08:00"
      value: 500
      days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]

  results:                             # Written back after optimization completes
    generated_at: "2026-05-10"
    method: nsga2
    n_solutions: 12
    elapsed_seconds: 87.3
    pareto_front:                      # All non-dominated solutions (flow-style, one per line)
      - {x: [0.60, 0.60, 0.60], f: [62.1, 48.3]}
      - {x: [0.80, 0.80, 0.80], f: [64.8, 46.2]}
      - {x: [1.00, 1.00, 1.00], f: [66.9, 43.8]}
    reference:                         # Researcher-selected representative point (not unique optimum)
      x: [0.80, 0.80, 0.80]
      f: [64.8, 46.2]
      regimen:
        protein_intake:
          "Restriction phase":  0.60
          "Maintenance phase":  0.80
      objectives:
        muscle_mass: 64.8
        gfr: 46.2
```

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
- **`reference` is not `best`**: Multi-objective optimization has no unique optimum. `reference` is a researcher-selected representative point from the Pareto front, chosen to illustrate a specific tradeoff. Users should inspect the full `pareto_front`.
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

A K×4 Regimen is expressed in `simulation.schedules` as multiple entries for the same variable, each with a distinct `date_range` (one entry per segment):

```yaml
simulation:
  schedules:
    - variable: protein_intake      # segment 1: t₁=day 0, d₁=60 days, p₁=daily, v₁=0.6
      time: "08:00"
      value: 0.6
      date_range: "2026-01-01 ~ 2026-03-01"
      label: "Restriction phase"
    - variable: protein_intake      # segment 2: t₂=day 60, d₂=90 days, p₂=daily, v₂=0.8
      time: "08:00"
      value: 0.8
      date_range: "2026-03-02 ~ 2026-06-29"
      label: "Relaxation phase"
    - variable: protein_intake      # segment 3: t₃=day 150, d₃=215 days, p₃=daily, v₃=1.0
      time: "08:00"
      value: 1.0
      date_range: "2026-06-30 ~ 2026-12-31"
      label: "Maintenance phase"
```

A solved K×4 Regimen from optimization is serialized in `optimizer.results.reference.regimen` (see §6):

```yaml
optimizer:
  results:
    reference:
      regimen:
        protein_intake:
          "Restriction phase":  0.60
          "Relaxation phase":   0.80
          "Maintenance phase":  1.00
```

This YAML is human-readable and directly exportable as iCal: each segment maps to one VEVENT with DTSTART, DURATION, RRULE (daily), and DESCRIPTION (dose value).

---

## 8. Game Conversion Block (LM Game Extension)

The optional `game:` block defines conversion hints for transforming an LM Scenario into a card-based educational game. This is a first-class extension in LM format 1.0.

```yaml
game:
  title: "CKD: The Protein Paradox"
  narrative: >
    You are managing your diet and exercise to preserve kidney function,
    but every gram of protein you add risks accelerating kidney decline.
  
  variable_mapping:
    gfr:
      display_name: "Kidney Function"
      icon: "kidney"
      low_threshold: 30               # Below this triggers warning card
      critical_threshold: 15          # Below this triggers game-over condition
    muscle_mass:
      display_name: "Muscle Strength"
      icon: "muscle"
      low_threshold: 60               # Percentage of baseline
  
  input_mapping:
    protein_intake:
      card_type: "player_action"
      low_value: {label: "Low Protein Diet", value: 0.6}
      high_value: {label: "High Protein Diet", value: 1.2}
    
  formula_mapping:
    gfr_decline:
      card_type: "environment_event"
      trigger_condition: "gfr < 30"
      event_name: "Kidney Crisis"
      severity: major

  conversion_rules_version: "0.1"
```

LM Game conversion rules (how YAML state/input/formula maps to card mechanics) are specified in a separate document: `LM_GAME_CONVERSION_0.1.md`.

---

## 9. Model Library Standards

An LM file is eligible for submission to the **LM Open Model Library** if it meets all of the following:

### 9.1 Required Fields

- `metadata.name`, `metadata.version`, `metadata.description`
- `metadata.authors` (at least one author with name)
- `metadata.citation` (a preferred citation string; DOI strongly recommended)
- `metadata.sources` (at least one entry for each literature-derived parameter)
- `metadata.license` (must be CC BY 4.0 or more permissive for Library inclusion)
- `metadata.lm_format_version: "1.0"`

### 9.2 Validation

A Library-eligible LM file must pass validation by an LM-compatible engine:
- All variable references in formulas resolve to declared variables
- All import paths resolve
- Initial values are within declared bounds
- No circular imports

### 9.3 File Naming Quality Markers

The LM Reference Library uses a three-suffix convention to track file quality status. Files without any of these suffixes are considered validated and publication-ready.

| Suffix | Meaning | Gitignored |
|--------|---------|------------|
| `_nosim` | Simulation cannot run (YAML parse error, unresolved variable reference) | Yes |
| `_noopt` | Simulation passes; optimizer block present but fails | Yes |
| `_noref` | Missing literature sources (`TODO:SOURCE` present) | Yes |

The combination `_nosim_noopt` is the default initial state for untested files. When a file passes both `--sim` and `--opt`, its suffix is removed and it enters the clean state.

The deprecated suffixes `_mw` (pure component) and `_TODO` (draft) have been removed; their semantics are now expressed exclusively through the three-marker system.

An optional YAML field `metadata.reviewed: true` may be set by the author to indicate that mechanisms and parameter magnitudes have been manually verified. This is not a publication gate and does not affect gitignore behavior.

### 9.4 Contribution License Agreement

By contributing an LM file to the LM Open Model Library, contributors confirm:
1. They have the right to submit the contribution.
2. They agree to license their model contribution under CC BY 4.0.
3. The contribution does not reproduce verbatim text from copyrighted materials.

---

## 10. Versioning

### 10.1 LM format Version History

| Version | Date | Status | Notes |
|---------|------|--------|-------|
| 1.0 | 2026-05-10 | Draft | Initial release |
| 1.0 | 2026-05-19 | Draft | Updated to align with Reference Engine: `evidence:` as separate top-level block with 8 sub-types; `simulation.schedules` updated to flat-list pulse format; `simulation.plans` added; `optimizer:` moved to top-level; `optimizer.results.reference` (replaces `best`); `metadata.description` structured form; `simulation.step_size` removed (only in `metadata.step_size`) |
| 1.0 | 2026-05-22 | Draft | Added `optimizer.start_date` / `end_date` / `step_size` (optional evaluation time window, §6.1); optimizer schedule tiers T2/T3/T4 introduced |
| 1.0 | 2026-05-23 | Draft | Formalized Sim / Opt independence principle (§6 intro) |
| 1.0 | 2026-05-28 | Draft | **Breaking**: `optimizer.inputs` deprecated → unified `optimizer.schedules` (ADR 0088); T2 field `optimize.time: ["HH","HH"]`; T3 field `optimize.days_pool + days_n` (backend enumerates combinations, replaces explicit `days_options`); T4 field `optimize.date_range: [[lo,hi],[lo,hi]]` (two mandatory windows); all legacy fields (`time_window`, `opt_step`, `days_options`, `date_start_window`, boolean flags) removed from engine and all YAMLs |
| 1.0 | 2026-05-24 | Draft | Added `optimizer.mc.seed` (optional integer; fixed = reproducible MC, omit = random each session); reproducibility note added to §2.3; Reference Engine returns `session_seed` in API response |
| 1.0 | 2026-06-06 | Draft | **ADR 0096**: Added §9.3 File Naming Quality Markers — three-suffix convention (`_nosim`, `_noopt`, `_noref`); deprecated `_mw` and `_TODO`; added `metadata.reviewed` optional field. |
| 1.0 | 2026-06-05 | Draft | **ADR 0092**: `type: input` variable `unit` field must be a bare event-quantity unit (e.g. `mg`, `g/kg`, `kcal`, `MET-h`); rate units (`mg/day`, `kcal/day`, `g/kg/day`) are prohibited — they belong in `description` or `reference`; continuous rate processes must be modeled as `parameter` with step-multiplied formulas. **ADR 0045 update**: `simulation.mc` and `optimizer.mc` are now separate, fully independent blocks; `optimizer.mc.enabled` and `optimizer.mc.sim_runs` deprecated → use `runs:` in each block; `algorithm.seed` controls only NSGA-II and does not fall back as mc.seed. §5.3 added for `simulation.mc`; §6 mc block updated. |

### 10.2 Versioning Policy

LM format uses semantic versioning:
- **Patch** (1.0.x): Clarifications, editorial fixes, no schema changes
- **Minor** (1.x.0): Backward-compatible additions (new optional fields)
- **Major** (x.0.0): Breaking schema changes

LM files declare `metadata.lm_format_version` to indicate which version of the specification they target. LM-compatible engines should accept files targeting older LM format versions.

### 10.3 Planned Extensions (LM format 1.1+)

- Validation metadata (uncertainty ranges, sensitivity indices)
- Model composition constraints (required/prohibited imports)
- Multi-individual simulation (household, cohort scenarios)
- Probabilistic event modeling (stochastic state transitions)

Features already implemented in Reference Engine (backported into LM format 1.0): `evidence:` block, `simulation.plans`, `optimizer.results`, flat-list `simulation.schedules`.

---

## 11. Reference Implementation

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
4. Execute formulas in the declared priority order with correct step-scaling
5. Support K×4 Regimen as an optimization input formalism
6. Serialize outputs including solved Regimen in the format defined in §6.4

---

## 12. Citation

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

## 13. Governance

LM format 1.0 is authored and maintained by Fan Shen. The specification is intended to evolve as a **shared research commons** — meaning:

- The format itself is public and open.
- Anyone may implement an LM-compatible engine.
- Contributions to the format specification may be proposed via the public repository.
- Commercial products may implement or support the LM Format; the LM name should primarily identify the open specification, model library, and scholarly community.

> LM is intended to function as a shared research commons rather than an exclusive commercial brand. Commercial products may implement or support the LM Format, but the LM name should primarily identify the open specification, model library, and scholarly community.

---

*Life Matters Format (LM format) 1.0 — Initial draft, Fan Shen, 2026-05-10*  
*Sun Yat-sen University, School of Systems Science and Engineering*
