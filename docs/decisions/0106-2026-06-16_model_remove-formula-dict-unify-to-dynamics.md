# ADR 0106 - Removing the formula: Dict Form, Unifying Variable Updates to dynamics:

**Date**: 2026-06-16
**Status**: Adopted
**Supplements**: ADR 0105 (the precondition for `formula:` dict's conditional `step_unit` requirement no longer exists)

---

## Background

LM format historically allowed a `formulas:` entry to update a variable in two ways:

```yaml
# Form A: a dynamics: dict (with or without step)
dynamics:
  fitness: fitness + (g * training_load - k1 * fitness) * step

# Form B: a formula: dict (a static algebraic assignment, overlapping in meaning with dynamics)
formula:
  performance: p0 + fitness - fatigue
```

Both directly update a model state variable and are semantically equivalent, differing only in whether `step` is used. This caused:

- A modeler had to choose between the two forms, but the deciding criterion, whether `step` is involved, was implicit.
- The `formula:` keyword carried two entirely different meanings: a dict form (updating a variable) and a string form (stored into formula_results).
- ADR 0105 had to carve out a special exemption from the required-`step_unit` rule just for `formula: dict`, adding rule complexity.

---

## Decision

Remove the `formula:` dict form. Every operation that directly updates a state variable now uses `dynamics:` uniformly, whether or not `step` is involved:

```yaml
# Correct, unified form: dynamics: used for every variable update
dynamics:
  fitness:     "fitness + (g * training_load - k1 * fitness) * step"   # involves step
  performance: "p0 + fitness - fatigue"                                  # does not involve step (a static algebraic form)
```

`formula:` keeps its string form (meaning: compute an intermediate value and store it into `formula_results`, without writing it back to a model variable):

```yaml
formula: "p0 + fitness - fatigue"   # stored into formula_results['formula_name'], does not update any variable
```

---

## Field Semantics After Disambiguation

| Field | Type | Semantics |
|------|------|------|
| `dynamics: {var: expr}` | Dict | Directly updates a model state variable, supporting an expression with or without step |
| `formula: "expression"` | String | Computes an intermediate value, stored into `formula_results` (does not update a variable) |

The two are mutually exclusive within the same formula entry (only one may be chosen).

---

## Migration

Every `formula: {dict}` entry in every existing model has been uniformly replaced with `dynamics:`. The replacement is purely mechanical: the field name `formula:` becomes `dynamics:`, with the sub-content unchanged. Every replaced entry contained no `step`, so `step_unit` is not needed (ADR 0105's rule does not trigger).

---

## Impact

| File | Change |
|------|------|
| Existing YAML models | `formula: {dict}` to `dynamics:` (bulk-completed) |
| `model.md` | The schema example updated; the `step_unit` note's exemption reference for `formula: dict` removed; the `lm_score` irreversible-mode example updated |
| ADR 0105 | The table's `formula: dict` row changed to a `dynamics: without step` row; a note added to the background section |
| `validator.py` | Optional: remove the dict-branch check for the `formula:` value type (can be kept during a backward-compatibility period, but new use is not recommended) |
