# Regimens and the Optimizer

## simulation.plans[*].regimens: a Time-Driven input Sequence

The only legal location for a simulation's input schedule is `simulation.plans[*].regimens` (ADR 0109; the field name is set in ADR 0117). Each plan holds a set of regimen entries describing the time-driven input for each `type: input` variable in that plan. Every input is sustained (ADR 0127): each entry corresponds to a `[time_start, time_end)` effective window, and each step that falls inside the window is written as `value / N_steps`, with a step outside the window automatically 0, so that each hit (each matching day) independently accumulates a contribution always equal to `value`, regardless of `step_size` and regardless of how many days `days`/`date_range` cause this entry to match (ADR 0131; see "value semantics" below). There is no literal "pulse mode" that exists separately from sustained; a window narrowed to a single step is numerically identical to what used to be called a pulse.

The top-level `simulation.schedules` field is no longer supported (the old format, deprecated in ADR 0109).

### Standard Format (a Single-Plan Model)

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2026-01-04"
  plans:
    - id: default
      label: "Baseline plan"
      regimens:
        - variable: carb_intake
          time_start: "07:00"
          value: 50.0
          days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]   # optional; absent = every day
          date_range: ["2026-01-01", "2026-01-04"]        # optional; absent = the whole run
          label: "Breakfast carbs"
        - variable: carb_intake
          time_start: "12:00"
          value: 80.0
          label: "Lunch carbs"
        - variable: carb_intake
          time_start: "18:30"
          value: 60.0
          label: "Dinner carbs"
```

### Field Reference

| Field | Type | Required | Description |
|------|------|------|------|
| `variable` | string | Yes | Must be the name of a `type: input` variable in `variables` |
| `time_start` | `"HH:MM"` | — | The start of the effective window (24-hour clock); see "window-width default rules" below for the default |
| `time_end` | `"HH:MM"` | — | The end of the effective window; see "window-width default rules" below for the default |
| `value` | number | Yes | The total quantity within the window (not the per-step quantity); the accumulated contribution always equals this number, regardless of step size or window width |
| `days` | `[Mon...Sun]` | — | A list of three-letter abbreviations; absent means it triggers every day |
| `date_range` | `["YYYY-MM-DD", "YYYY-MM-DD"]` | — | The entry is effective only within this calendar range; absent means the whole span from `start_date` to `end_date` |
| `label` | string | — | Explanatory text for GUI display |

#### Window-Width Default Rules (ADR 0127)

Neither `time_start` nor `time_end` is required. Leaving both unwritten does not mean "configuration is missing"; it means the modeler has chosen a specific window width, which the engine resolves by the following rules (the modeler does not need to piece together `time_start`/`time_end` manually, just state the intent clearly):

| Form | Effective window | Applicable scenario |
|---|---|---|
| Neither written | All day, `["00:00", "24:00"]` | A day-rate input, such as a total daily caloric deficit or an average daily intake, a quantity that never had a concept of "the moment it happens" and should not be forced to invent a trigger time that does not exist |
| Only `time_start` written | `time_end` = `time_start` (a single-step window) | A discrete event (a meal, a dose), numerically identical to what used to be called a "pulse" |
| Both written | An explicit interval | An input spanning multiple steps in a sub-day step-size model, such as sustained intensity or sustained protection |

Do not hand-write `time_start == time_end` just to reproduce the pulse effect; writing only `time_start` already achieves it, and writing two identical values explicitly makes readers more likely to think the two are independently adjustable.

### Multiple Entries Versus Multiple Cycles

The same variable can have multiple entries (three meals, for instance), and the engine sums every hit within the same step:

```yaml
# Three meals: at most one hits per step, so the accumulated result equals a single meal's value (staggered across times)
# With a 1-day step size: all three entries hit within the same step, so dietary_protein = 0.27+0.27+0.26 = 0.80
```

`date_range` expresses a staged plan (such as progressive training phases); do not substitute "listing repeated weekly entries" for it:

```yaml
# Correct: use date_range to distinguish phases
regimens:
  - variable: training_load
    time_start: "09:00"
    value: 50.0
    days: [Mon, Tue, Wed, Thu, Fri]
    date_range: ["2026-01-01", "2026-01-28"]   # base phase, 4 weeks
  - variable: training_load
    time_start: "09:00"
    value: 100.0
    days: [Mon, Tue, Wed, Thu, Fri]
    date_range: ["2026-01-29", "2026-02-25"]   # build phase, 4 weeks

# Wrong: listing week by week (redundant, entry count = number of weeks x 2)
regimens:
  - variable: training_load
    value: 50.0
    date_range: ["2026-01-01", "2026-01-07"]   # week 1
  - variable: training_load
    value: 50.0
    date_range: ["2026-01-08", "2026-01-14"]   # week 2 (identical to week 1, pointless)
```

### Recommended Form for a Multi-Phase Plan: Baseline Plus Increment (ADR 0126)

The engine handles multiple regimen entries for the same variable by resetting to zero each step and accumulating them one by one, not by overriding. That behavior itself is correct; it is exactly how the three-meals accumulation above is designed to work. But if a plan is written as "one complete target value per phase, with mutually exclusive switching achieved end-to-end through `date_range`," every entry has to get its `date_range` exactly right, and missing one on a single entry makes it effective for the entire run, stacking on top of every other phase (a real case: a three-phase diet plan's restriction-phase entry was missing `date_range`, so its target quantity kept stacking after the plan moved into later phases, systematically inflating the symptom score).

The recommended alternative is one baseline entry with no `date_range` (which should be effective for the whole run anyway, so there is no "forgot to set an end date" trap) plus several increment entries scoped by `date_range`, whose value is the difference relative to baseline, not an absolute target:

```yaml
# Correct: baseline plus increment; the one entry with no date_range is meant to be effective for the whole run anyway
regimens:
  - variable: fodmap_intake
    time_start: "00:00"
    value: 22.0                              # baseline: the maintenance-phase target, effective for the whole run
  - variable: fodmap_intake
    time_start: "00:00"
    value: -15.0                             # the restriction phase's reduction relative to baseline
    date_range: ["2026-01-01", "2026-01-28"]
  - variable: fodmap_intake
    time_start: "00:00"
    value: -2.0                              # the reintroduction phase's reduction relative to baseline
    date_range: ["2026-01-29", "2026-04-01"]

# Wrong: writing a full target value for each phase; missing or misdated date_range on any one entry causes stacking instead of replacement
regimens:
  - variable: fodmap_intake
    value: 7.0
    date_range: ["2026-01-01", "2026-01-28"]   # the restriction-phase target
  - variable: fodmap_intake
    value: 20.0
    date_range: ["2026-01-29", "2026-04-01"]   # the reintroduction-phase target
  - variable: fodmap_intake
    value: 22.0                                 # the maintenance-phase target: if this entry's date_range is
                                                 # forgotten, it also applies during the other two phases and stacks with them
```

With the baseline-plus-increment form, even if an increment entry's `date_range` is missing, it only overcounts the increment for a few days, a small deviation of the same order of magnitude, rather than a full-target-value-scale stacking error; that is why this form is preferred over writing a complete target value for every phase.

#### A Known Limitation on the opt Side: When Phase-Switch Timing Is a Search Variable, an Increment Entry's `date_range` Cannot Track It

T4 (intervention date-range optimization, covered below) can turn a phase-switch time point itself into a search variable. But if a multi-phase plan is expressed in the baseline-plus-increment style and one increment entry's `date_range` endpoint should "follow the switch timing T4 searches for," the engine currently does not support this kind of cross-entry date linkage; `date_range` can only be a hardcoded date, or that entry's own `optimize.date_range` search window, and it cannot reference "the value another regimen entry's search found":

```yaml
# T4 searches for "which day the restriction phase ends," but the reintroduction phase's
# increment entry cannot automatically follow that search result for its date_range start
regimens:
  - variable: fodmap_intake
    value: 22.0                       # baseline
  - variable: fodmap_intake
    optimize:
      value: [-18.0, -10.0]
      date_range:                      # T4: the restriction phase's own duration is the search variable
        - ["2026-01-01", "2026-01-01"]
        - ["2026-01-22", "2026-03-05"]
  - variable: fodmap_intake
    value: -2.0
    date_range: ["???", "2026-04-01"]  # wrong: the start cannot be written as "the end date T4 found above"
```

Current handling (accepted as a controlled cost, not an engine defect): a need for multi-phase date linkage on the opt side falls back to manually fixing the `date_range` search window, letting the modeler confirm the result on the Sim side and, if needed, manually adjust the opt input's date range before re-searching, rather than pursuing a single optimization run that automatically links every phase boundary. The convenience of this kind of opt-side input design is left for a future feature improvement or plugin and is not within the current scope of investment (ADR 0126).

### Relationship to GUI inputEvents

`simulation.plans[*].regimens` is the source of GUI `inputEvents` at model load time: the GUI parses the YAML per plan and populates `inputEvents`, and from then on a simulation session actually uses `inputEvents` (which the user can edit and which an optimization result can overwrite). There is no runtime conflict of "the YAML overriding the GUI's edits on every step" (ADR 0074/0115).

Note for modelers: a manual adjustment to a variable in the GUI generally takes effect directly; if that variable is also marked as a decision variable via `optimization.startpoint.regimens` (an `optimize:` block), the optimizer takes over that variable's value while running, as an independent search/preview state separate from the Sim panel's manual value.

### Discrete Inputs Do Not Need a Zero-Value Point

A discrete `type: input` quantity (a meal, a dose, etc.) does not need an inserted `value: 0` closing point; it is automatically 0 outside the window.

```yaml
# Correct: only write the nonzero moment
- variable: carb_intake
  time_start: "07:00"
  value: 50.0

# Wrong: a redundant zero-value point
- variable: carb_intake
  time_start: "07:30"
  value: 0.0    # unnecessary; it is automatically zero outside the window
```

Exception: a genuinely continuous-rate variable, such a sustained infusion's `infusion_rate`, needs an explicit closing point.

### Backward Compatibility: the Old Dict Format

The old dict format (`{varName: {interpolation, points: [{time: seconds, value}]}}`) still parses in the engine but is no longer recommended; new models should use the flat list format.

---

## simulation.plans: Predefined Multi-Plan Comparison

`simulation.plans` lets a modeler pre-configure several named plans in the YAML, which the GUI presents directly as a Plan list for parallel multi-plan simulation (F-MPLAN) when the model loads.

Use cases:
- Paper models (papers/): write a Pareto front's representative points as named plans, so a reader can open the model and immediately compare, say, kidney-protection-first versus muscle-preservation-first.
- Clinical controls: pre-configure a "guideline standard dose" plan alongside an "optimized dose" plan, showing the exact inputs a paper's figures correspond to.

Format:

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2026-12-31"
  plans:
    - id: "kidney_protect"
      label: "Kidney-protection-first (Pareto endpoint)"
      regimens:
        - variable: dietary_protein
          time_start: "08:00"
          value: 0.22
          days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
          label: "Breakfast protein"
        - variable: dietary_protein
          time_start: "12:00"
          value: 0.21
          label: "Lunch protein"
        - variable: dietary_protein
          time_start: "18:00"
          value: 0.22
          label: "Dinner protein"
    - id: "balanced"
      label: "Clinically balanced plan"
      regimens:
        - variable: dietary_protein
          time_start: "08:00"
          value: 0.29
          label: "Breakfast protein"
        - variable: dietary_protein
          time_start: "12:00"
          value: 0.27
          label: "Lunch protein"
        - variable: dietary_protein
          time_start: "18:00"
          value: 0.28
          label: "Dinner protein"
    - id: "muscle_preserve"
      label: "Muscle-preservation-first (Pareto endpoint)"
      regimens:
        - variable: dietary_protein
          time_start: "08:00"
          value: 0.38
          label: "Breakfast protein"
        - variable: dietary_protein
          time_start: "12:00"
          value: 0.36
          label: "Lunch protein"
        - variable: dietary_protein
          time_start: "18:00"
          value: 0.37
          label: "Dinner protein"
```

Field reference:

| Field | Type | Required | Description |
|------|------|------|------|
| `id` | string | Yes | The plan's unique identifier (lowercase with underscores) |
| `label` | string | Yes | The name displayed in the GUI |
| `regimens` | list | Yes | Same entry format as above; each entry is one time-driven input event |

The only legal location (ADR 0109): a simulation's input schedule may only exist under `simulation.plans[*].regimens`, never under a top-level `simulation.schedules`. A single-plan model uses one plan with `id: default`.

Session semantics of a Plan: a Plan is a GUI runtime object, and what the modeler pre-configures in the YAML is only its initial state; the user can go on adding, editing, or deleting plans in the GUI, and none of that is written back to the YAML file.

---

## Layered Constraints

1. Model: may only `import` other Models, and must never reference a Story.
2. Story: composes Models and configures a scenario, and may include `optimization` configuration and `patches`.
3. Cycle detection: `LoaderEngine` automatically blocks a circular import.

---

## lm_score: Life Matters's Core Healthy-Duration Metric

`lm_score` is the Life Matters framework's conventional core variable, representing the accumulated duration for which key metrics simultaneously satisfy a healthy condition. It is an ordinary `state` variable plus a standard equation, written out in full by the modeler in the YAML, with no special engine handling. The variable name `lm_score` is a convention and can be freely overridden or renamed.

### Two Accumulation Semantics

| Semantics | Description | Applicable scenario |
|------|------|---------|
| Recoverable (cumulative) | Accumulates while the condition holds and pauses while it does not; resumes accumulating once it holds again | Chronic-disease management, a hypoglycemic episode that can be survived, mild symptoms |
| Irreversible (latch) | Once the condition stops holding, the `lm_alive` flag zeroes out permanently, and accumulation never resumes even if the condition is later restored | Organ failure, an irreversible death event |

### YAML Form

Recoverable mode (the recommended default):

```yaml
variables:
  lm_score:
    type: state
    value: 0.0
    unit: day
    description: "Healthy duration: accumulated simulated days with both GFR and blood pressure in the safe range"
    reference: "Life Matters Framework core metric"

equations:
  lm_score_update:
    step_unit: day    # lm_score's unit is day, so step_unit must be declared as day.
                      # If simulation.step_size is hour and this is mistakenly left as hour,
                      # step accumulates by the hour and inflates lm_score by 24x (a real incident that has been observed).
    dynamics:
      lm_score: "lm_score + step if (GFR >= 15 and SBP <= 160) else lm_score"
    description: "Accumulates healthy duration (recoverable)"
```

Irreversible mode (latch, suited to death or organ failure):

```yaml
variables:
  lm_score:
    type: state
    value: 0.0
    unit: day
    description: "Healthy duration: accumulated days before the first collapse (irreversible)"
    reference: "Life Matters Framework core metric"
  lm_alive:
    type: state
    value: 1.0
    description: "Survival flag: 0 = irreversible collapse, 1 = alive"

equations:
  lm_alive_check:
    condition: "not (GFR >= 15 and SBP <= 160)"
    dynamics:
      lm_alive: "0.0"                    # once triggered, permanently 0
    description: "Detects collapse and latches the survival flag"
  lm_score_update:
    step_unit: day    # same as above: must match lm_score's day semantics, not copied from another equation's hour
    dynamics:
      lm_score: "lm_score + lm_alive * step"
    description: "Accumulates healthy duration (irreversible)"
```

### As an Optimization Objective

```yaml
optimization:
  objectives:
    - variable: lm_score
      metric: final          # the accumulated healthy days at the end of the simulation
      direction: maximize    # maximize healthy duration
```

### Merging Across Multiple Model Imports

When several submodels each define `lm_score` with a different condition, importing them lets the later one override the earlier one (following the standard import override rule). To AND multiple submodels' conditions together, the modeler explicitly rewrites the `lm_score_update` equation in the top-level model:

```yaml
# Top-level model: explicitly merges Model A's GFR condition and Model B's SBP condition
equations:
  lm_score_update:
    step_unit: day
    dynamics:
      lm_score: "lm_score + step if (GFR >= 15 and SBP <= 160) else lm_score"
```

### Design Principles

- `lm_score` is an ordinary variable, fully transparent, and its value at every simulation step can be output and inspected.
- The condition expression uses the same asteval sandbox as any equation and can reference any variable in the model.
- Multiple health conditions can be freely combined with `and`/`or`.
- In the GUI's objective-variable selector, `lm_score` is marked with a star for easy identification, with no other special behavior.

---

## optimization: Decision Variables and Schedule Optimization

### The Separation of Sim and Opt

`simulation:` and `optimization:` are independent scenario descriptions, but the GUI can convert between them:

| Field/concept | simulation | optimization |
|----------|-----------|-----------|
| Time range | `simulation.start_date`/`end_date` | `optimization.start_date`/`end_date` (optional) |
| Step size | `simulation.step_size` (required) | `optimization.step_size` (optional, defaults to sim's) |
| Monte Carlo | — | `optimization.mc` |
| Fixed input plus decision variable | `simulation.plans[*].regimens` (for visualization) | `optimization.startpoint.regimens` (a unified list) |

Fallback: when an `optimization.*` field is missing, the engine inherits it from the corresponding `simulation.*`; the GUI clearly labels the source ("from sim" versus "overridden").

GUI conversion:
- "Import from Sim": copies the Sim tab's current inputEvents into `optimization.startpoint.regimens` as decision variables, with bounds inferred automatically.
- "Send to Sim": pre-fills a Pareto-recommended solution's regimen as Sim inputEvents.

### optimization.startpoint.regimens: a Unified List of Decision Variables and Fixed Background Quantities

`optimization.startpoint.regimens` is a unified list of decision variables and fixed background quantities (ADR 0109). An entry with an `optimize:` block is a decision variable; an entry without one is a fixed background quantity. The `startpoint` block describes which initial protocol the optimizer starts its search from.

```yaml
optimization:
  startpoint:
    regimens:
      - variable: metformin_dose      # a fixed background quantity (no optimize block)
        time_start: "08:00"
        value: 500
        days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
        label: "Metformin baseline dose (background)"
```

Independence: the optimizer runs its own internal simulation on every evaluation, using only the events decoded from `optimization.startpoint.regimens`, and never reads `simulation.plans[*].regimens`; the two paths never affect each other (see `optimizer_eval.py` in the `life-matters-reference-engine` repository). `optimization.startpoint.regimens` must be defined explicitly; there is no implicit fallback (ADR 0109 removed the fallback chain).

### The Evaluation Time Window (start_date / end_date / step_size)

The optimizer runs one internal simulation on every evaluation, and its time range and step size can be independent of the GUI's visualization settings:

| Field | Type | Description |
|------|------|------|
| `start_date` | `"YYYY-MM-DD"` | The optimization evaluation's start date; defaults to `simulation.start_date` |
| `end_date` | `"YYYY-MM-DD"` | The optimization evaluation's end date; defaults to `simulation.end_date` |
| `step_size` | `{value, unit}` | The evaluation step size; defaults to `simulation.step_size` |

Design principles:
- All three are optional; when not declared, they inherit from simulation.
- Declaring them explicitly guarantees reproducibility: when a YAML file with `optimization.results` is published, a reader can rerun the optimization with the same time window.
- The evaluation step size is recommended to match `simulation.step_size`; if the model's dynamics timescale allows, it can be coarsened somewhat to speed up the search.
- The GUI's time controls (the toolbar's date and step-size fields) are passed into the engine as an `optimizer_override` when running an optimization, taking priority over the YAML's static values.

Typical usage (shortening the evaluation window to speed up the search):

```yaml
simulation:
  start_date: "2026-01-01"
  end_date:   "2030-12-31"   # 5-year visualization

optimization:
  start_date: "2026-01-01"
  end_date:   "2027-12-31"   # evaluate over only 2 years to speed up the search
  step_size:
    value: 1
    unit: day
```

---

The optimizer splits a regimen's parameterized search into four granularity tiers, ordered by scientific value and computational complexity:

| Tier | Optimized object | Variable type | Typical scenario |
|------|---------|---------|---------|
| T1 | An event value (dose or intensity) | A continuous real number, optionally discretized with `value_step` | A drug dose, a nutrient intake amount |
| T2 | An event's timing (within a time window) | A discrete integer (a time-slot index) | An eating window, dosing timing, circadian rhythm |
| T3 | A day-of-week combination (freely chosen from candidate days) | A discrete integer (a combination index) | Exercise frequency, fasting-day scheduling |
| T4 | The intervention's start date (within a date window) | An integer (a day offset) | Treatment timing, seasonal intervention |

Each `inputs` entry can independently enable any combination of tiers; the x vector is the concatenation, in order, of every enabled dimension.

T1's `optimize.value` searches by default within the continuous `[lo, hi]` interval, and the solution can carry arbitrary decimal precision; once the optional `optimize.value_step` is declared, the engine snaps the continuous solution during decoding onto a grid starting at `lo` with a spacing of `value_step`, which suits a scenario that needs clinically or practically readable increments, such as a feeding volume in 5 mL steps or a metabolic equivalent in 0.1 MET-h steps; when it is not declared, behavior is unchanged and the search remains continuous.

### T2: Time-Window Optimization

T2 builds on the unified `time_start`/`time_end` interval fields (previous section). `optimize.time_start` searches for the interval's start; the interval's width (`time_end - time_start`) stays fixed by default (1 dimension, the most common case). If `optimize.time_end` is additionally declared, the interval's end is also searched independently (2 dimensions).

1 dimension: searching the start with a fixed width (this covers both a narrow window with zero width, "at what time does it trigger," and a wide window, "starting when, with an unchanged duration"):

```yaml
regimens:
  - variable: meal_carbs
    time_start: "08:00"
    time_end: "08:00"            # width = 0 (a single-step window); stays 0 after the search
    days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]   # fixed days of the week (T3 not active)
    label: "Breakfast carbs"
    optimize:
      value: [30, 80]
      time_start: ["07:00", "09:00"]   # the start's search window [lo, hi]
      time_step: "1h"                  # optional; defaults to 1h; a finer scenario can use 15min
```

```yaml
regimens:
  - variable: care_intensity
    time_start: "08:00"
    time_end: "20:00"             # sustained, width = 12h
    label: "Daytime care intensity"
    optimize:
      value: [0.0, 288.0]
      time_start: ["06:00", "10:00"]   # the start is searched within [06:00,10:00], width stays 12h
```

2 dimensions: the start and end are searched independently (the interval's width is itself a decision variable):

```yaml
regimens:
  - variable: care_intensity
    time_start: "08:00"
    time_end: "20:00"
    label: "Daytime care intensity (both start and end searched)"
    optimize:
      value: [0.0, 288.0]
      time_start: ["06:00", "10:00"]   # the start's search window
      time_end: ["18:00", "22:00"]     # the end's search window (independent of the start)
```

- `optimize.time_start` and `optimize.time_end` are both formatted as `["HH:MM", "HH:MM"]` (24-hour clock, inclusive of both endpoints).
- Legal values for `time_step` are `"1h"` (default) and `"15min"`, applied to both windows at once. The engine expands them into discrete time slots, so `["07:00","09:00"]` with `1h` becomes `["07:00","08:00","09:00"]` (3 slots).
- Writing only `optimize.time_start` (with no `optimize.time_end`) gives 1 dimension: the searched `time_end` equals the searched `time_start` plus a fixed width (equal to that entry's own `time_end - time_start`, which is 0 for a single-step window).
- Writing both `optimize.time_start` and `optimize.time_end` gives 2 dimensions: both ends are searched independently, with no linkage between them.
- The old field `optimize.time` is deprecated and no longer supported; use `optimize.time_start` instead.
- Scientific significance: in chronobiology, chrono-nutrition and chronopharmacology, the timing of an intervention is itself a key decision variable, and this framework brings it explicitly into the optimization search space.

### T3: Day-of-Week Combination Search

```yaml
regimens:
  - variable: exercise_load
    time_start: "17:00"
    label: "Exercise"
    optimize:
      value: [30, 90]
      days_pool: [Mon, Tue, Wed, Thu, Fri, Sat]  # the candidate set of days
      days_n: [3, 5]                             # choose 3 to 5 days from the pool
```

- `days_pool`: the candidate set of days (three-letter abbreviations, Mon-Sun).
- `days_n: [min, max]`: internally enumerates every legal combination from the pool satisfying min ≤ n ≤ max, encoded as an integer decision variable.
- When T3 is active, the top-level `days:` field is not written (no fixed days).

### T4: Intervention Date-Range Optimization

```yaml
regimens:
  - variable: caloric_restriction
    time_start: "08:00"
    days: [Mon, Tue, Wed, Thu, Fri, Sat, Sun]
    label: "Caloric restriction"
    optimize:
      value: [400, 800]
      date_range:                                # exactly two groups, both required
        - ["2026-05-01", "2026-05-30"]           # the start date's search window [lo, hi]
        - ["2026-12-31", "2026-12-31"]           # the end date's search window (write the same date twice to fix it)
```

- `optimize.date_range` must contain exactly two groups: the first is the start date's search window, and the second is the end date's search window.
- If the end date is fixed, write `["YYYY-MM-DD", "YYYY-MM-DD"]` with both values the same.
- When T4 is active, the top-level `date_range:` field is not written (no fixed date range).
- Typical scenarios: treatment intervention timing, a seasonal intervention window, the timing of post-disaster relief resource deployment.

> The `mode: sustained` plus `time_range` form (ADR 0098) was superseded by ADR 0100 and is no longer supported by the engine; old YAML files need to switch to the unified `time_start`/`time_end` interval fields below.

#### Conceptual Foundation: Extensive and Intensive Quantities in value/delivery/days/date_range

When a person describes an intervention plan in ordinary language, the numeric value itself belongs to one of two kinds of physical quantity: a total that stays fixed once summed, however it is internally split up in execution, such as "100 g of protein per day"; or a state level that should be maintained, independent of how it is split up or repeated, such as "an infusion rate of 10 mL per hour." This distinction corresponds to the categories of extensive quantity (additive under subdivision) and intensive quantity (unchanged under subdivision) in physics, and it operates at two levels of granularity in LM format:

- Within a day, across simulation steps: how `value` is distributed among the simulation steps within the same matching day is decided by the `delivery` field; see "delivery: total | level" below.
- Across matching days: when `days`/`date_range` select several matching days, whether `value` should be redistributed by the number of matching days is decided by the accounting-domain design; see "value semantics" below.

| Granularity | Extensive (changes with subdivision/aggregation) | Intensive (unchanged by subdivision/aggregation) |
| --- | --- | --- |
| Within a day, across simulation steps | `delivery: total` (default): `value` is split evenly across `N_steps`, with the accumulated contribution always equal to `value` | `delivery: level`: `value` is delivered as-is to every hit step, with no division |
| Across matching days | Not offered: LM format has no mode for "spreading a total `value` across multiple matching days" | The only mode: each matching day independently delivers the full `value`, unaffected by how many days match (ADR 0131) |

The two levels of granularity share the same extensive/intensive vocabulary but are not symmetric: `delivery` offers two options at the within-day level, and the modeler declares one explicitly based on the variable's physical meaning; there is no corresponding "extensive" option at the across-day level, since `days`/`date_range` only select which days trigger and never take part in any across-day redistribution. This design keeps `value`'s reading from drifting whether the simulation step size is refined or the number of matching days changes, provided the downstream equation is written at its native granularity, that is, its `step_unit` is no coarser than `simulation.step_size`; for the specific anti-pattern when this precondition does not hold, see "the delivery decision rule" below.

#### value Semantics: Each Matching Day Independently Delivers the Full Amount, Adaptively Split by N_steps (ADR 0131, Superseding 0099)

`value` (and the bounds of `optimize.value`) represents the total quantity within a single hit window, one matching day, the same dimension as a pulse's "one-time total," and it is also delivered independently and in full on every matching day, neither diluted because `date_range`/`days` cause this entry to match more days, nor concentrated because it matches fewer. This satisfies two invariants at once: `step_size` only affects precision (the principle from the Euler discrete integration section), and "how many days matched" only affects the total number of deliveries, not how much each delivery carries, since the latter is exactly the definition of what a "daily rate" modeling intent means.

At load time, the engine precomputes, for every sustained entry:

```
N_steps = the duration of a single hit window / step_size
```

At run time, every hit step is written as `value / N_steps` (still following the rule that a `type: input` is not multiplied by `step`, so the dynamics equation needs no change). Each matching day's accumulated contribution equals `(value / N_steps) x N_steps = value`, independent of `step_size`; the total simulation duration, the number of days `date_range` covers, and the day-of-week filtering by `days` only decide how many days this entry independently delivers `value` on, and never change what `value` itself means. A zero-width window (a single-step window, formerly called a pulse) is simply the special case `N_steps=1`, following the exact same rule as any other window width, not a second, different mechanism.

The duration of a single hit window looks only at that entry's own `time_start`/`time_end` (defaulting to all day, 24h), and never at how many days `date_range` covers or how many weekdays remain after `days` filtering; the latter two are only filters for whether to trigger on a given day, exactly as they have always been mere filters for a pulse event and never a factor in the pulse's own value.

> When modeling, fill in `value` as "how much this entry contributes on each hit (each day)," for example writing `value: 50` for "a training load of 50 per day," with no need to manually calculate the plan's total run length and multiply it in; only when `value` genuinely expresses a budget-style semantics, "the total stays fixed at this number no matter how many days actually match," does the modeler need to control, outside the YAML, that `date_range` or the simulation's total duration no longer changes (in that case `value` is playing the role of "a manually fixed total budget spread thin," which the engine does not distinguish; whether this effect is needed, and whether the total budget should be adjusted whenever the number of matching days changes, is left entirely to the modeler's judgment. The old ADR 0099/0126 semantics of "the total stays fixed" has been superseded by this ADR and is no longer the engine's default behavior; achieving that effect now requires the modeler to simply avoid changing the number of matching days).

Example: the same modeling intent, "a fixed daily rate" (now the default, and the only, behavior):

```yaml
regimens:
  - variable: training_load
    time_start: "00:00"
    time_end: "24:00"
    value: 50.0            # 50 units per day, regardless of how many days this plan actually runs
    date_range: ["2026-01-01", "2026-02-01"]  # 32 days: total delivered = 50 x 32 = 1600
    label: "constant daily training load"
```

Shortening `date_range` to 16 days (everything else unchanged) makes the total delivered `50 x 16 = 800`; it is still 50 per day, and only the day count changed, with the total scaling proportionally, which is exactly the behavior a daily rate should have. (By contrast, the old ADR 0099-era behavior locked the total instead, doubling the effective daily rate to 100 as the number of days shortened; this has been superseded by this ADR.)

#### delivery: total | level: Splitting a Total Versus a Constant Level (ADR 0132)

The previous section's "each day independently delivers the full amount" solves one problem, that a total should scale proportionally with the number of matching days, but it does not solve a different problem: some `type: input` variables' physical meaning is not "how much was put in during this period" at all, but "what state or setting currently holds," such as tonight's sleep duration, bedtime timing, or care intensity. Such a variable is never accumulated in an equation; it is read as an instantaneous value directly, compared against a baseline or used as a multiplicative factor, and it should stay unchanged throughout the entire effective window. ADR 0099/0131's "total divided by N_steps" splitting rule is wrong for this kind of variable, since the reading would drift inversely with `step_size` (and even with the number of matching days), and a model would silently become inaccurate the moment it was run differently, such as a step-size variation for a convergence check, or being imported into another model with a different `step_size`. This problem has existed since ADR 0098 first proposed sustained (motivated precisely by a variable such as sustained care intensity that should stay constant) and since ADR 0099 redefined `value`'s semantics that same day; it simply had never surfaced in a variable-step-size scenario.

The test: is this variable accumulated downstream (its contribution to a total growing over time), or read directly (compared against some baseline, or used as a coefficient, where the reading at a given moment should not differ just because `step_size` changed)?

| `delivery` | Semantics | Delivered amount per hit step | Basis for the choice |
|---|---|---|---|
| `total` (default, omit to use it) | The window's total for each matching day, split per the previous section's rule | `value / N_steps` | Training load, total food intake, total dose delivered, quantities that are inherently "how much was put in or consumed during this period," accumulated downstream |
| `level` | A constant level, not split | `value` itself | Sleep duration, bedtime timing, a diet-quality score, care intensity, protection level, quantities that are inherently "the current setting or state," read downstream as an instantaneous value for comparison or multiplication |

```yaml
regimens:
  - variable: sleep_hours
    time_start: "00:00"
    time_end: "24:00"
    value: 6.0          # directly the target level, no need to manually multiply by N_steps
    delivery: level       # each hit step delivers 6.0 directly, unaffected by step_size or the number of matching days
    days: [Mon, Tue, Wed, Thu, Fri]
    label: "Weekday sleep duration"
```

`delivery: level` is a no-op for a single-step window (a pulse), since a pulse is already the special case `N_steps=1`, and dividing or not dividing gives the same result. `days`/`date_range`/`time_start`/`time_end` mean exactly the same thing under either `delivery` value, only deciding whether a given day triggers, and never affecting the amount delivered.

This is not implemented through a variable-naming convention (such as an internal suffix on the variable name that makes the engine treat it specially); that route was already explicitly rejected while discussing a `consumed_by` allowlist scheme (the existing principle that variable names are free and the engine recognizes no reserved names). `delivery` is a declaration written explicitly on the regimen entry, at the same level as `time_start`/`time_end`/`days`, not a naming convention, and not a reintroduction of the pulse/sustained switch that ADR 0100 removed; that switch was genuinely redundant, since a single window-width number already determines point versus window with no information lost by removing it, whereas `total`/`level` is a new, independent dimension: Banister's `training_load` and this section's `sleep_hours` use exactly the same window width (all day), so window width alone cannot distinguish the two, and an explicit declaration is required.

#### The delivery Decision Rule: a Structural-Position Test, Also Catching the Day-Lumped-Map Anti-Pattern (ADR 0133)

"Accumulated downstream versus read as a coefficient" is a first-order test, but it is not precise enough and can mask a deeper problem: some variables appear on the surface to "need level," but the real reason is not that variable's own physical nature; it is that the downstream equation itself was written as a coarse-grained (such as `step_unit: day`), one-shot map rather than a step-by-step integrable dynamics equation. In that case, `delivery: level` merely redistributes "the answer computed once for the day" repeatedly to every finer step; the reading itself does not drift, but the equation never converges to a more precise solution as the step size is refined, which is a weaker form of consistency than genuine reading invariance and should not be conflated with true step-by-step dynamics.

Rule A: does this variable participate in the downstream equation as a coefficient or an instantaneous state, in an equation written at native step granularity (with a `step_unit` no coarser than `simulation.step_size`)?
- If yes, use `delivery: level`, which is the only correct, final answer requiring no further treatment. Examples: `training_intensity`/`pace` (read natively minute by minute), `care_intensity`/`self_protection`/`rest_hours` (read natively hour by hour); such variables physically have no "total" dimension at all, so there is no ambiguity.
- Is the variable itself "how much was put in this time," accumulated by a downstream state? Then use `delivery: total` (the default).

Rule B (the day-lumped-map anti-pattern test): if a variable, in order to display a "level" effect, forces its downstream equation to use a `step_unit` coarser than `simulation.step_size` (such as `day`) to compute the net change across multiple simulation steps in one shot, and then relies on `delivery: level` to redistribute this one-shot answer, unchanged, to every finer step, this is a design-smell signal that cannot be fixed by adjusting `delivery`; it means the variable was modeled with the wrong primitive to begin with. The correct direction is to split it into an instantaneous indicator readable at native granularity (a true level) and a derived state accumulated from it (a true total, following the same pattern as `lm_score`), rewriting the equations as genuine ODEs at their native `step_unit` accordingly.

> Known hits: `sleep_hours`/`bedtime_hour` (in `sleep_schedule_sim.yaml`/`burnout_allostatic_sim.yaml`, where `sleep_pressure_dynamics` declares `step_unit: day` yet uses an all-day sustained input with `delivery: level`, repeatedly reading the same daily quantity hour by hour); `nutrition_score` is suspected of the same pattern but has not been individually verified. The fix, splitting it into an `is_asleep` state plus a pulse pair in place of a duration-type input, is outside the scope of this document; see ADR 0133 for detail.

#### When to Use a Fixed-Width Sustained Input Versus a Pulse Trigger Plus a Decaying State (ADR 0126/0127/0131)

`N_steps` is precomputed by the engine at load time (before the simulation starts), not determined dynamically at run time, which means an entry's own `[time_start, time_end)` window width must already be a determinate number at the time the YAML is written (as of ADR 0131, how many days `date_range`/`days` covers no longer enters this calculation, since it is only a hit filter, so it no longer needs the "total duration" to be a determinate number, only "how wide this one window is"). Judge accordingly:

- Is this input's own `[time_start, time_end)` window width already determinate at load time? If yes (an accurate window width can be computed without depending on the optimizer's search result, such as a hardcoded `"08:00"~"20:00"`, or a filter condition like `days`/`date_range` that only affects which days trigger and never the window width itself), write it normally per ADR 0127's default rule or as an explicit interval, with no extra mechanism needed.
- If no (the window width itself is a T2/T3/T4 search variable, such as "what time each day work ends" with both endpoints to be searched, or it depends on a state that can only be determined at run time), a fixed-width form cannot work, since `N_steps` cannot be computed at load time; switch instead to a pulse trigger (a single-step window) feeding into a `type: state` decaying variable that downstream equations read. This is a purely local, step-by-step recurrence mechanism (each step only needs the current state plus the current input) and does not need to know in advance how many steps will run, so it is not limited by an unknown window width.

### The Unified Interval Representation: time_start / time_end (ADR 0100/0127)

Any input's effective window is the same pair of `[time_start, time_end)` fields taking a value on the timeline, not a choice among several mutually exclusive "modes"; the only difference is window width:

| Relationship between `time_start` and `time_end` | Meaning |
|---|---|
| Neither written | All day, `["00:00","24:00")` (ADR 0127's default rule, for a day-rate input) |
| `time_end == time_start` | A single-step window, `N_steps=1`, with `value` written into that step as-is (formerly called a "pulse") |
| `time_end != time_start` (not spanning the whole day) | Each step within the interval gets `value/N_steps` (the previous section's equation) |
| `time_start="00:00"`, `time_end="24:00"` | `[0,24)` full coverage, the value taken when the window width is the whole day, identical to the result of "neither written," not a separate state |

```yaml
regimens:
  - variable: care_intensity
    time_start: "08:00"
    time_end: "20:00"           # a [08:00, 20:00) interval, sustained
    date_range: ["1945-08-06", "1945-08-11"]   # only affects which days trigger, not the value below
    label: "Daytime care intensity"
    optimize:
      value: [0.0, 36.0]        # each matching day's window total; 12h / step=1h gives N_steps=12
```

#### Window Width Is a Quantity Along One Axis; Decay Is a Downstream Concern Independent of It

Inputs of different window widths (a single step, all day, an explicit interval) are the same thing, `value`, the total within the effective window, split evenly across `N_steps` hit steps, taking different values along the window-width axis; they are not several mutually exclusive "input types" (ADR 0127). Every window width shares the same unit convention (a bare unit with no time denominator; see "the unit convention for input variables" above), and `value`'s dimension does not change with width.

Decay is not part of the "window width" axis's discussion at all; decay is a matter of whether a downstream `type: state` variable's own `dynamics` equation should include a term for "naturally falling off over time," an independent equation dimension, unrelated to how wide the input driving it happens to be: a state driven by a wide-window input can just as well need decay, for instance a plasma concentration continuing to fall off by its half-life after a sustained infusion rate stops.

A narrow window (a single-step window) has a trap of its own that a wide window does not have: a narrow window writes a nonzero value only in the one step it hits, and every other step reads 0 for that variable; if a downstream equation reads this variable directly and evaluates more frequently than the window occurs, for instance `step_size: hour` while the window only hits once a day, that equation reads 0 in 23 of every 24 evaluations. If the equation is trying to express "is a certain sustained state currently in effect" rather than "did this one-time event happen today," it gets misled by this illusion of "zero the rest of the time" (two such cases were found on 2026-07-09, one each in `hypertension_gout_sim.yaml` and `smoking_stress_sim.yaml`; see ADR 0126). A wide window has no such trap, since it is nonzero at every step within its entire effective window; this is also one reason ADR 0127 set "writing no time at all means all day" as the default: a day-rate input defaults to a wide window, avoiding this trap at the source, and only a genuinely discrete event, such as a meal or a dose, needs a narrow window, and whenever a narrow window is needed the modeler has explicitly written `time_start`, rather than the engine choosing it on their behalf.

The test for the narrow-window case (look at which structural position this variable occupies in the equation, not whether it is multiplied by `*step`):

An easy trap to fall into is assuming "whether this term is multiplied by `step`" is the deciding factor. It is not: `step` is identically 1 whenever the equation's `step_unit` matches `simulation.step_size` (`step = step_size_sec / equation_step_sec`), so multiplying by `*step` or not is numerically equivalent in that case and produces no dilution or amplification either way; `*step` only marks whether this quantity needs conversion across a `step_unit`, unrelated to whether this input can be read at all. The real test is which structural position this input variable occupies in the equation:

- Allowed: appearing inside some `state`'s own `dynamics` as a top-level added or subtracted term (whether or not that term is further multiplied by `*step`), that is, a self-referencing accumulation of the form `state: state + ... ± f(input variable) ...`. Examples: the term `eta_abs*cigarettes_per_day` in `nicotine_plasma: nicotine_plasma + eta_abs*cigarettes_per_day - k_nic*nicotine_plasma*step`, `body_weight: body_weight - caloric_deficit/7700`, and the ketone-spike term in `uric_acid: uric_acid + ... + max(0, psi_ketone*caloric_deficit/700 - 15.0)*step` (even though it is multiplied by `*step` here, since `step_unit` matches `simulation.step_size` and `step` is identically 1, this is completely equivalent to not multiplying and remains a legal top-level added term). This pattern is "when a one-time event fires, accumulate its contribution into the variable's own running ledger," with the state itself remembering the accumulated result and no need to re-read the raw input every step.
- Not allowed: appearing inside a "target value" or "relaxation target" sub-expression (the target-value part inside `rate*(state - target)*step`), or appearing in a pure algebraic snapshot equation with no self-referencing `+ itself` term. Both structures express "what is the current state" or "where is the system converging to," which can only be constructed from `state`/`parameter`, because this kind of computation implicitly assumes the quantity is roughly stable across adjacent steps, whereas a narrow-window input is 0 for 23 of 24 steps and suddenly nonzero for one, making the target value flicker violently between "completely inactive" and "fully in effect," which breaks the premise of relaxation dynamics. The problem is not that "reading 0 is wrong" (reading 0 is fine in itself; 0 is exactly what the value should be when it has not triggered); the problem is the structural mismatch of using a violently flickering quantity to play the role of what should be a stable target value. Both known bugs (the relaxation target in `bp_dynamics` and the algebraic snapshot in `health_economic_index`) fall precisely into this category; every known correct precedent (`nicotine_plasma`/`thiazide_level`/the ketone term in `uric_acid`, etc.) is a top-level self-referencing added term. A wide-window input, such as the "how much was smoked today" that `health_economic_index_update` needs, is far less likely to fall into this trap than a narrow-window input, which is exactly where ADR 0127's "all-day default" and this structural test complement each other.

> The "structural position" test above currently can only be checked by the modeler (the engine has no validator for it yet), and it is known to miss cases. An alternative approach has been planned (not yet implemented): declaring a `consumed_by: [equation_name, ...]` allowlist on every `type: input` variable, with the engine validating that this input is referenced only by the equations on the allowlist. This solves a different class of problem, reading the wrong physical quantity (such as `bp_dynamics` needing to read `body_weight` but reading `caloric_deficit` instead, which no window-width adjustment can fix and only an allowlist can catch), complementary to ADR 0127 (the window-width default rule, which solves the class of problem where the window is too narrow and another equation reads a false 0), not two versions of the same solution.

Deprecated fields (old YAML files need to update manually; the engine no longer reads the old fields):

| Old field (no longer supported) | Equivalent new form |
|---|---|
| `time: "HH:MM"` (pulse) | `time_start: "HH:MM"` (omitting `time_end` defaults to the same value, i.e. a pulse) |
| `mode: sustained` plus `time_range: [a, b]` | `time_start: a, time_end: b` |
| `mode: sustained` (with no `time_range`) | `time_start: "00:00", time_end: "24:00"` |

Write `time_start`/`time_end` directly, with no need to first decide whether a single point, an interval, or all day is wanted; the three are simply different positions of the same pair of fields on the timeline, not three separate switches or branches. The `days` (day-of-week filter) and `date_range` (calendar interval) fields are unchanged and orthogonal to `time_start`/`time_end`.

### x-Vector Encoding Rules

The x vector is expanded in the order of the `optimization.startpoint.regimens` list, with each entry contributing dimensions in the order `[value?, time_start?, time_end?, days?, date_start?, date_end?]`:

| Tiers enabled on the entry | x dimensions contributed | Variable type |
|--------------|-----------|---------|
| T1 only | 1 (value) | A continuous real number |
| T2 only, 1 dimension (`time_start` only) | 1 (time_start_idx) | An integer |
| T2 only, 2 dimensions (`time_start` plus `time_end`) | 2 (time_start_idx, time_end_idx) | Two integers |
| T1 + T2 (1 dimension) | 2 (value, time_start_idx) | A real number plus an integer |
| T1 + T2 (2 dimensions) | 3 (value, time_start_idx, time_end_idx) | A real number plus two integers |
| T1 + T3 | 2 (value, combo_idx) | A real number plus an integer |
| T1 + T4 | 2 to 3 (value, date_start_offset[, date_end_offset]) | A real number plus one or two integers |
| A fixed background quantity (no optimize) | 0 | — |

A mixed-integer vector is handled through NSGA-II's continuous relaxation; a single-objective algorithm (L-BFGS-B / Nelder-Mead) does not support integer variables, so enabling T2/T3/T4 automatically switches to NSGA-II with a warning.

Example: `meal_carbs` (T1 plus T2, 1 dimension) and `exercise_load` (T1 plus T3) each contribute 2 dimensions, giving an x length of 4:

```
x = [carbs_value, time_start_idx, exercise_value, combo_idx]
    [   55.3,           1,            62.0,            2   ]
# time_start_idx=1 -> slots[1] = "08:00"; time_end = "08:00" + the fixed width
# combo_idx=2      -> combinations(pool, n)[2] = [Mon, Wed, Fri]
```

`optimization.results.recommended` stores only the raw `x`/`f` vectors, not a decoded, human-readable result; decoding is a pure algorithmic derivation from `x` plus the structure of `optimization.startpoint.regimens`, needing no additional persistence (see the `optimization.results` section below).

---

## optimization.results: the Embedded Format for Optimization Results

Once an optimization finishes, the result is written back into an `optimization.results` block, stored in the same YAML file alongside the configuration. This means publishing a model also publishes its results; when a model with results loads, the Opt panel's "continue from here" checkbox is on by default, and the user can choose a warm start or a cold start.

### Full Structure

```yaml
optimization:
  method: nsga2
  objectives: [...]
  startpoint: {...}
  algorithm: {...}

  results:                          # written by the GUI once optimization finishes; no need to fill it in by hand
    generated_at: "YYYY-MM-DD"     # the generation date (the date part of ISO 8601)
    method: nsga2                  # the algorithm used
    n_solutions: 8                 # the number of solutions on the Pareto front
    elapsed_seconds: 87.3          # this run's elapsed time in seconds
    pareto_front:                  # every non-dominated solution (flow style, one solution per line)
      - {x: [0.30, 0.29, 0.30], f: [65.8, 47.1]}
      - {x: [0.35, 0.33, 0.34], f: [66.9, 44.8]}
    recommended:                    # a point the modeler has flagged as recommended from the Pareto front (not a unique optimum)
      x: [0.30, 0.29, 0.30]       # decision-variable values (in the order of the decision entries in optimization.startpoint.regimens)
      f: [65.8, 47.1]             # objective-function values (in the order of objectives)
```

### Field Reference

| Field | Type | Description |
|------|------|------|
| `generated_at` | A date string | The date written, used to judge whether the results are stale |
| `method` | string | The algorithm name (nsga2 / l-bfgs-b / nelder-mead) |
| `n_solutions` | int | The number of solutions on the Pareto front |
| `elapsed_seconds` | float | This run's elapsed time |
| `pareto_front` | list | Every non-dominated solution, each element `{x: [...], f: [...]}` |
| `recommended.x` | list | The recommended point's decision-variable values (chosen by the modeler from the Pareto front, not a unique optimum) |
| `recommended.f` | list | The recommended point's objective values |

A decoded, human-readable plan or objective dictionary is no longer stored; both the human-readable display and the "send to Sim" feature decode from `x`/`f` on the fly (the frontend's `xToInputEvents`), avoiding the maintenance of two parallel formats (a `recommended.regimen`/`recommended.objectives` dictionary once existed, and was removed on 2026-06-21 since no code path ever read it and the save logic had long since stopped generating it).

Mapping between the x vector and inputEvents: the x vector is expanded in the order of the decision entries, those with an `optimize:` block, in `optimization.startpoint.regimens`, with each entry contributing dimensions per its enabled tiers: T1 contributes 1 continuous-real dimension (value), and T2/T3/T4 each contribute 1 integer dimension (a time-slot index, a combination index, or a day offset). This mapping is implied by the structure of `optimization.startpoint.regimens` and needs no additional storage; the frontend's `xToInputEvents` function parses it in the same order (see `sim_design.md`).

### Design Principles

- Full overwrite of `results`: each save completely replaces the old `results` with the new front, keeping no history; a Pareto front only ever improves or holds steady, never regresses.
- Uniform format: `pareto_front` uses YAML flow style (`{x: [...], f: [...]}` on one line), so 50 solutions means 50 lines, without harming the model's readability.
- Warm or cold start (the user's choice): the Opt control bar's "continue from here" checkbox is always visible, selectable (a warm start) when results already exist, and disabled (a cold start) when there are none. If the objective function, a constraint, or a decision variable's search range is changed after checking warm start, the checkbox turns orange with a "continuing from here may match less well" warning, but is not forced to switch to a cold start.
- Sim reads opt results: loading a model that has `recommended.x` prompts the Sim panel to ask whether to pre-fill the recommended point as the current inputEvents; the user can choose to load it or ignore it.
- Opt-to-Sim, many-to-many: a Pareto front is N groups of inputs; the software reassembles each of the N Pareto solutions into a valid Sim inputEvents (a Plan) for F-MPLAN's parallel simulation and comparison; opt.results keeps only the raw x/f vectors.
- `recommended` does not represent a unique optimum: multi-objective optimization has no single "best solution," and `recommended` is a balance point the modeler has flagged, which the user should weigh against the rest of `pareto_front` themselves. The name avoids `reference` so as not to be confused with `variables.<name>.reference`/`equations.<name>.reference` (the literature-citation fields).
- Published means results included: once the modeler runs the optimization, saves the model, and uploads the YAML, a recipient sees the Pareto front and the recommended point the moment they open it; `results` can be read on its own.
- No results is also valid: `optimization.results` is an optional block; a model without this field runs normally, starting its search from a random initial population.

### Workflow

```
Modeler                          GUI                          Model file
  |                              |                              |
  |-- opens a model with results ->  | shows the historical Pareto front  |
  |                              | toolbar: this model has historical results |
  |-- clicks "run optimization" --> | warm start (historical solutions as the initial population) |
  |                              | evolves for n more generations |
  |-- clicks "save results to model" --> | POST /api/optimizer/write-results
  |                              |-------------------------->  | optimization.results overwritten
  |-- clicks "download model" --> | GET /api/file-raw/{path}     |
  |   receives the .yaml file    |                              |
```
