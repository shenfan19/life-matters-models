# models/references — Source Model Components

## Purpose

`references/` holds **the foundational physiological/social dynamics submodels that each paper scenario depends on**, the base component layer of the entire model ecosystem.

Each file describes a single physiological system or mechanism (renal function, exercise fatigue, PK/PD, etc.), for an upstream scenario (`papers/`, `scenarios/`) to compose via the `imports:` mechanism.

**What does not belong here**:
- A complete, runnable simulation scenario (put it in `papers/` or `scenarios/`)
- Game content (put it in `stories/`)

## Directory structure

```
medical/
  disease/      Chronic-disease progression models (CKD, diabetes, hypertension, etc.)
  fitness/      Exercise adaptation and fatigue (Banister, etc.)
  medicine/     Pharmacokinetics (PK/PD, first-order absorption/elimination)
  nutrition/    Nutrient metabolism (protein, energy, water)
  physiology/   Basic physiology (glomerular filtration, muscle synthesis, ALT dynamics, etc.)
  psychology/   Mental health and cognition models
  surgery/      Perioperative surgical models
social/
  conflict/     Conflict and stress models
  demography/   Demographic dynamics
  economy/      Economic income/expenditure models
  law/          Legal and policy constraints
  psychology/   Mental health and cognition models
  technology/   Technology-diffusion models
environmental/
  climate/      Climate and environmental dynamics
risk/
  actuarial/    Actuarial and mortality models
```

## Usage

```yaml
# Referenced via imports in a scenario file under papers/ or scenarios/
imports:
  - medical/fitness/banister_fitness_fatigue
  - medical/disease/ckd_renal_filtration
```

The loader recursively merges imported submodels, with a same-named variable/equation in the root file overriding the submodel's definition.

## Validation status (sim validation)

Every "independently analyzable reference model" (a model containing a `type: input` variable, see the "references/ directory convention" section of `docs/authoring/imports_and_organization.md`)
should have multiple `simulation.plans`, cross-checking simulation curves against each other to verify the model's own directional correctness, rather than just running `--sim` once and calling it done if nothing errors. Method: write 2-3 scenarios for the same model that should mechanistically produce different, predictable-direction outcomes,
run `--sim` and check each curve matches the expected direction; once that passes, run `--opt` to confirm the front is non-degenerate (not all solutions crowded onto one point), and only then remove the `nosim`/`noopt` markers from `metadata.todo`.

**Models that have completed this round of validation** (2026-07-06):

| Model | # of plans | Validation focus |
|------|---------|---------|
| `medical/physiology/banister_fitness_fatigue_2026.yaml` | 3 | The adaptation/fatigue two-time-constant mechanism; found and fixed a model-level bug where overly narrow `bounds` held performance at 0 permanently |
| `medical/disease/chronic/ckd_protein_muscle_2026.yaml` | 3 | The directional tradeoff between low-protein renal protection and high-protein muscle preservation; found and triggered the investigation of the engine-level bug described below |
| `medical/nutrition/diet/mediterranean_diet_2026.yaml` | 3 | Adherence and the red-meat antagonistic effect; corrected an LDL rate-constant conversion error (3 years mistakenly computed as 7); the opt front honestly degenerates to a single point, since the only decision variable is monotonically beneficial for LDL but has no effect on CRP, so there is no genuine conflict between the two objectives, not a bug |
| `medical/fitness/individual/running_2026.yaml` | 3 | The pace-lactate-fatigue-performance coupling; found an engine bug where "a single-day model only ran for 1 hour," and changed the optimization objective from "final performance" to "calories burned" to remove the degenerate front |
| `social/demography/population/population_growth_2026.yaml` | 3 | Fertility policy affecting population structure through the birth rate; the model originally used `step_unit: month` (unsupported by the current format), rewritten to a day-level step |
| `social/economy/labor/labor_economic_2026.yaml` | 3 | The income-fatigue-productivity tradeoff among overtime, vacation, and skill investment; the model originally used `step_unit: week` (unsupported by the current format) and was completely missing `simulation.plans`, both now supplied |

Whether the remaining models under `references/` still carry the `nosim`/`noopt` markers in `metadata.todo` keeps changing as modeling progresses;
before checking each one the same way, do not assume its `optimization.results` or `description.result` values are trustworthy —
trust the live result from running the query command below instead, not the historical numbers in this document.

**A note for batch publishing**: this project's sole criterion for "publishable" is whether `metadata.todo` exists/is non-empty (ADR 0120, independent of the filename).
Before a batch publish, it's advisable to also run `--sim`/`--opt` on files whose `metadata.todo` is empty but that have not appeared in any validation-round table, to confirm they aren't affected. Query command (run under `models/references/`, requires PyYAML):

```python
import yaml, glob
for f in sorted(glob.glob('**/*.yaml', recursive=True)):
    data = yaml.safe_load(open(f, encoding='utf-8'))
    if isinstance(data, dict) and not (data.get('metadata') or {}).get('todo'):
        print(f)
```

### Representative examples: what sim validation exercised in LM

3 representative ones (other similar cases are not listed individually):

1. **`running_2026.yaml`'s `threshold_intervals` plan**: verified the combination of sustained-interval "window total" semantics
   (ADR 0099, when `time_start` differs from `time_end` the `value` is spread across the steps within the window) with the nonlinear threshold mechanism —
   at RPE=8, a 4.5min/km pace, sustained for 60 minutes, lactate rises from 0.5 to 18.5 mmol/L (well past the 4 mmol/L threshold zone), and athletic performance collapses from
   100 to 2.1 points, exactly matching the model's own documented description of "a marked decline after 40-50 minutes."
2. **`ckd_protein_muscle_2026.yaml`'s three-arm comparison** (0.6 / 0.8 / 1.2 g/kg/day): verified the core method of "multiple plans
   cross-confirming a model's directional correctness" — the lower the protein intake, the higher the final GFR (43.06 to 42.01) but the lower the final muscle mass
   (24.36 to 35.28kg), the two curves strictly opposite, matching the model's own described core clinical tradeoff; this was also the original case in this round that uncovered the pulse-
   reset clamp bug (this model previously carried no `nosim`/`noopt` marker and appeared "already validated," when in fact an engine bug had silently contaminated it).
3. **`labor_economic_2026.yaml`'s `skill_investment` plan**: verified the "short-term cost, long-term
   benefit" intertemporal tradeoff across a multi-plan comparison — compared with the balanced plan, this plan has lower year-end savings (238627 versus 249399 yuan) but a doubled productivity coefficient
   (1.0 to 2.0), quantifying the core tension in the model's problem statement that "human-capital investment carries a short-term opportunity cost."

## Contribution guidelines

See [`docs/authoring/README.md`](../../docs/authoring/README.md). Every component file must include:
- `metadata.description` (a mechanism explanation; see [`docs/authoring/description_writing.md`](../../docs/authoring/description_writing.md) for the writing convention)
- top-level `references` (literature sources)
- a `description` and `unit` for every variable

Every model file welcomes edits, additional parameter sources, or bug fixes from any user.
