# ADR 0105 - step_unit Made Conditionally Required, Deprecating the dt/step_size Dynamics Symbols

**Date**: 2026-06-16
**Status**: Adopted
**Revises**: ADR 0104 (the rule making `formulas.step_unit` required was too strict)

---

## Background

ADR 0104 required every formula to declare `step_unit`. After it was rolled out, this rule turned out to be too strict:

- A static algebraic formula (the `formula:` dict form, such as `performance: p0 + fitness - fatigue`) never uses `step` at all, so forcing it to declare `step_unit` is meaningless.
- A large number of static formulas in existing models (banister, bergman, etc.) started triggering validator errors as a result, blocking normal loading.

In addition, the original formula-symbol table (ADR 0046) once recorded `dt` and `step_size` as aliases for `step`, and some old models (such as `digestive_system`) used `dt` in their dynamics expressions. Now that `step` is the sole standard symbol, these aliases should be deprecated.

The `formula:` dict form (writing a key-value mapping directly under a `formula:` block for a static algebraic assignment) was once allowed and overlapped in meaning with `dynamics:`; it has since been removed entirely by ADR 0106: every variable update now uses `dynamics:` uniformly, and `formula:` keeps only its string form (stored into formula_results).

---

## Decision

### 1. `step_unit` becomes conditionally required

Rule: `step_unit` is required only when `step` (or the deprecated symbols `dt`/`step_size`) appears in the `dynamics` expression.

| Formula type | `step_unit` | Note |
|---------|-------------|------|
| `dynamics:` containing `step` | Required | Enforced by the validator |
| `dynamics:` without `step` (a pure algebraic assignment) | Not needed | No time stepping, no declaration needed |
| A `formula:` string (stored into formula_results) | Not needed | Does not update a model variable, carries no stepping semantics |

### 2. Deprecating `dt` and `step_size` as dynamics symbols

- `dt` and `step_size` are prohibited in a `dynamics` expression.
- The validator raises an error when it detects one (`is_valid = False`), at the same severity as a missing `step_unit`.
- Every `dt` in every existing model has been replaced with `step`.
- `model.md` updated: both are marked deprecated, and `step` is the sole standard symbol.

> Engine compatibility: `simulation.py` still injects `dt` and `step_size` into the symbol table (`_STEP_SYMS`), to support a transitional period for the very few models not yet migrated. This injection can be removed in a later version, once it is confirmed every model has been migrated.

---

## Engine Adaptation

| Module | Change |
|------|------|
| `validator.py` | In `validate_formulas()`: `step_unit` is checked only when `dynamics` contains `step`/`dt`/`step_size`; detecting `dt`/`step_size` raises an additional error |
| `model.md` | Updated the "symbols inside a formula" table; updated the `step_unit` requirement note to conditional |
| Existing YAML models | Fully migrated: `dt` to `step`; the extra `step_unit` removed from static formulas (optional, not mandatory) |

---

## Trade-Offs

Given up: uniformly requiring every formula to declare `step_unit` (semantically complete, but too strict).

Gained:
- Static formulas no longer trigger a false validator error.
- `step` becomes the sole step-size symbol, eliminating the confusion dt/step_size caused.
- A dynamics formula still requires `step_unit`, so cross-module import semantics are unchanged.
