# ADR 0040 - Medical Research Conclusion Types and YAML Variable Mapping: the evidence_param Subtype
**Date**: 2026-04-22
**Status**: Implemented (a merged approach, different from this document's original plan; see "Implementation Record" below)

---

## Background

LM's positioning is "turning a literature statistical conclusion into a runnable dynamics model" (see sim_requirements.md). The current `parameter` type carries two different kinds of value at once:

1. A dynamics coefficient: a number that goes directly into a formula (a PK rate constant, an equation slope, etc.), given directly by the literature.
2. An epidemiological effect size: OR, HR, Cohen's d, etc., which cannot go into a formula directly and must first be converted into an effective coefficient (a multiplier or a probability increment) before use.

The current design pushes the conversion responsibility onto the modeler, violating LM's ease-of-use principle of "modeling like filling in a form." A medical researcher should not need to know the OR-to-RR conversion formula to use LM.

### Clarifying the dynamics perspective

From the simulation engine's perspective, OR/RR/HR are all just "different statistical expressions of the same thing," and only the final probability or coefficient enters the engine. The conversion is a data-preprocessing problem, not a simulation problem. This conversion naturally belongs at the Loader layer (the Loader already handles YAML assembly), not as manual work for the modeler.

### A note on population SD

Converting an effect size (Cohen's d) needs the population SD (σ). LM is currently an individual simulation (one person over time), so σ is a fixed value from the reference study's population and must be supplied by the modeler from the literature. If LM supports population simulation (many individuals in parallel) in the future, the Loader could be changed to dynamically read the simulated population's live SD instead; the current design stays compatible with that.

---

## Decision

Add `evidence_param` as a 5th top-level variable type, dedicated to epidemiological effect sizes.

- The modeler fills in the raw literature value in the YAML (OR=1.65, HR=0.82, d=0.68).
- The Loader converts it automatically during assembly into an effective coefficient usable by the engine, stored as `_effective_value`.
- The Simulator reads only `_effective_value`, unaware of the original type.
- The raw fields are all kept, for traceability and review.

---

## The 5 Variable Types (Updated)

| Type | What the engine reads | What the modeler fills in | Purpose |
|------|---------|----------|------|
| `state` | `value` (updated over time) | An initial value | A state that evolves over time |
| `input` | `value` (user-adjustable) | A control quantity | A user intervention |
| `parameter` | `value` (constant) | The final coefficient | A dynamics coefficient going directly into a formula |
| `probability_constant` | `value` (constant) | A probability value | A random event's probability, run at its expected value |
| **`evidence_param`** | `_effective_value` (converted by the Loader) | **The raw literature value** | An epidemiological effect size (OR/HR/ES, etc.) |

---

## evidence_param YAML Format

```yaml
evidence_params:
  # -- Relative risk RR ----------------------------
  smoking_lung_cancer_rr:
    evidence_type: relative_risk
    value: 2.7
    unit: "RR"
    # Loader: _effective_value = 2.7 (used directly, no conversion)
    reference: "Doll & Hill (1950) BMJ"

  # -- Odds ratio OR (must convert when prevalence > 10%) --------
  obesity_diabetes_or:
    evidence_type: odds_ratio
    value: 1.65
    unit: "OR"
    baseline_prevalence: 0.23        # the control group's prevalence p0 (required)
    # Loader: RR = OR / ((1-p0) + p0xOR) = 1.65 / (0.77 + 0.38) = 1.48
    # _effective_value = 1.48
    reference: "..."

  # -- Hazard ratio HR (needs a paired baseline risk) ------------
  chemo_mortality_hr:
    evidence_type: hazard_ratio
    value: 0.82
    unit: "HR"
    baseline_rate_ref: chemo_baseline_mortality   # the name of a probability_constant variable (required)
    # Loader: _effective_value = baseline_rate x HR (the reference resolved during Loader assembly)
    reference: "..."

  # -- Absolute risk difference ARD ------------------------------
  statin_cvd_ard:
    evidence_type: absolute_risk_difference
    value: 0.012
    unit: "prob/year"
    # Loader: _effective_value = 0.012 (used directly, added to the probability)
    reference: "..."

  # -- Effect size, Cohen's d -------------------------------------
  exercise_fev1_effect:
    evidence_type: effect_size
    value: 0.68                      # Cohen's d (the raw literature value)
    population_sd: 0.5               # the reference population's SD, in the same unit as unit (required)
    unit: "L"
    # Loader: _effective_value = d x sigma = 0.68 x 0.5 = 0.34 L
    reference: "..."

  # -- Incidence rate (already expressible with probability_constant; shown here as an equivalent form) --
  annual_diabetes_ir:
    evidence_type: incidence_rate
    value: 0.05
    unit: "prob/year"
    # Loader: _effective_value = 0.05 (used directly)
    reference: "IDF Atlas 2021"
```

---

## Loader Conversion Rules

| `evidence_type` | Conversion formula | Required auxiliary field |
|----------------|---------|------------|
| `relative_risk` | `effective = value` | — |
| `odds_ratio` | `effective = OR / ((1-p0) + p0xOR)` | `baseline_prevalence` |
| `hazard_ratio` | `effective = baseline_rate x HR` | `baseline_rate_ref` |
| `absolute_risk_difference` | `effective = value` | — |
| `effect_size` | `effective = d x population_sd` | `population_sd` |
| `incidence_rate` | `effective = value` | — |
| `regression_coefficient` | `effective = value` | — |
| `pk_rate_constant` | `effective = value` | — |
| `pd_emax` / `pd_ec50` / `pd_hill` | `effective = value` | — |

---

## How It Is Used in a Formula

Once the Loader has finished converting, an `evidence_param` variable is used in a formula exactly like an ordinary `parameter`; the modeler only needs to use the variable name:

```yaml
formulas:
  lung_cancer_dynamics:
    dynamics:
      # smoking_lung_cancer_rr's _effective_value = 2.7 (filled in by the Loader)
      lung_cancer_risk: baseline_lung_cancer_risk * smoking_lung_cancer_rr * smoking_intensity * dt

  glucose_response:
    dynamics:
      # exercise_fev1_effect's _effective_value = 0.34 L (filled in by the Loader after conversion)
      fev1: fev1 + exercise_fev1_effect * exercise_intensity * dt
```

The modeler does not need to know whether the raw value was an OR or a d; the Loader has already turned it into a coefficient with the correct dimension.

---

## Loader Implementation Key Points

```python
def resolve_evidence_params(model):
    for name, ep in model.get('evidence_params', {}).items():
        etype = ep['evidence_type']
        raw = ep['value']

        if etype == 'odds_ratio':
            p0 = ep['baseline_prevalence']
            effective = raw / ((1 - p0) + p0 * raw)
        elif etype == 'hazard_ratio':
            baseline = model.probability_params[ep['baseline_rate_ref']].value
            effective = baseline * raw
        elif etype == 'effect_size':
            effective = raw * ep['population_sd']
        else:
            effective = raw  # relative_risk / ARD / IR / PK-PD

        ep['_effective_value'] = effective
        # inject into the runtime variable space for formulas to reference
        model.runtime_vars[name] = effective
```

---

## Backward Compatibility

- The existing `parameter` type continues to work, for a coefficient that goes directly into a formula.
- `evidence_params:` is a new top-level section; an existing YAML with no such section simply skips it.
- `probability_constant` continues to handle existing probability types such as IR/CFR; `evidence_param`'s `incidence_rate` is an equivalent, more explicit way to write the same thing.

---

## Out of Scope

- Full validation of `evidence_type` by the Loader (raising an error when a required auxiliary field is missing).
- Propagating the conversion result's uncertainty interval (a 95% CI).
- NNT (= 1/ARD): no new type is needed, since it can be computed directly from ARD.

---

## Implementation Record (2026-06-21)

The actual implementation deviated from this document's original plan in one respect: no independent `VariableType.evidence_param` was added.

- The top-level section is named `evidence:` (not this document's draft `evidence_params:`), its sub-field is `type` (not `evidence_type`), and the subtype abbreviations are `rr`/`or`/`hr`/`ard`/`cohens_d`/`ir`/`beta`/`pk` (not this document's full names such as `relative_risk`/`odds_ratio`). See `model.md`'s "Variable Types (3) + Top-Level evidence Conversion" for the complete definition.
- A converted variable's type remains `parameter`; no separate type was created. Two new fields, `evidence_type` (the original subtype) and `evidence_raw_value` (the literature value before conversion), serve as provenance markers, and the raw fields are indeed kept (fulfilling this document's requirement that "the raw fields are all kept, for traceability"; an earlier implementation once missed this and has since been fixed).
- Why no separate type was created: the inner-loop optimizer (Modeller) has not yet been implemented, so no code path currently auto-tunes a `parameter`, and this document's constraint that it should "never participate in any optimization" is currently a vacuous constraint that does not need type isolation to enforce. Once the Modeller is implemented, it only needs to skip any parameter where `evidence_type is not None`, with no need to preemptively expand the type set for an optimizer that does not exist yet.
- The GUI edit form has not been implemented, so the modeler needs to edit the YAML directly; no formal model currently uses evidence, only the test fixtures `models/test/test_evidence_*_HOLD.yaml`.
- A second deviation (added the same day): this document's original plan, and the first implementation above, both used an `_effective`/`_effective_value` suffix to distinguish "the converted variable." In practice this suffix turned out to be counterintuitive: the YAML only declares the name without the suffix, and the modeler had to know out of nowhere to add `_effective` when referencing it in `dynamics`/`formulas`, with nothing in the YAML content hinting at this. Changed to: the converted variable keeps the same name as the evidence entry, with no suffix added; if an evidence name collides with a variable already declared in `variables:`, the Loader raises an error directly (to avoid a silent overwrite). The 8 `_HOLD` test fixtures were renamed and revalidated to match.

## Implementation Record (2026-06-23): Automatically Wiring applies_to into dynamics

This document and the two implementation records above only solved the step of "converting into a coefficient"; how the converted result gets wired into a state variable's dynamics equation had, up to this point, been entirely hand-written by the modeler. A review (discussion notes: `2026-06-22_evidence-to-dynamics-discussion-notes.md`) found that of the 8 subtypes, `ir`/`ard`/`hr`/`rr`/`or` have exactly one unambiguous way to be wired in (all of them "use the converted coefficient as a rate, accumulated into some target state"), while `cohens_d`/`beta`/`pk`'s wiring is itself a modeling judgment (the transitional form, the regression structure, and the PK model structure are not unique), and can neither be nor will be auto-generated.

A "do as much as can be done" scheme was implemented: adding three optional fields to an evidence entry, `applies_to` (the target state), `step_unit` (`minute`/`hour`/`day`, the same constraint as `formulas.step_unit`), and `rate_unit` (the rate's natural time unit, declared by `ir`/`ard` themselves and read by `hr`/`rr`/`or` from the `ir`/`ard` entry `baseline_ref` points to). Once declared, the Loader automatically generates one `dynamics` entry in `_apply_model_data`; when not declared, behavior is completely unchanged. See `model.md`'s "Automatically Wiring into dynamics" section for the complete field table and constraints.

`Formula.step_unit` is not used directly to express year/week/month, because `validator.py`'s `valid_step_units` only accepts `minute`/`hour`/`day`; the ratio between `rate_unit` and `step_unit` is computed into a numeric coefficient and written directly into the generated dynamics expression string, without relying on `Formula.step_unit` itself to express a longer time unit.

Validation: each of the 5 `_HOLD` test fixtures (ir/ard/hr/rr/or) had a state added that `applies_to` generated, compared step by step against a state from hand-written dynamics (ir/ard/hr use the same baseline, with values matching step by step; rr/or, whose hand-written example uses a different baseline than the one used to demonstrate applies_to, do not match numerically but were each checked correct with an independent calculation); the three fixtures cohens_d/beta/pk were unaffected (applies_to unsupported, no YAML change made). All 4 error paths (an unsupported subtype, a duplicate target declaration, rr/or missing baseline_ref, an applies_to target left undeclared) were verified to raise the correct error message.
