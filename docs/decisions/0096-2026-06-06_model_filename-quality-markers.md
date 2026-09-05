# 0096 - The Model-Filename Quality-Marker Convention

**Date**: 2026-06-06
**Status**: Superseded by [0101](0101-2026-06-14_model_hold-suffix-todo-field.md) (the three suffixes `_nosim`/`_noopt`/`_noref` were fully migrated to `_HOLD` plus `metadata.todo` on 2026-06-14)
**Category**: Model library management / engineering convention

---

## Background

As the `models/` directory grew to over 145 YAML files, three management problems surfaced:

1. Unclear publication boundary: which files should be published to GitHub? gitignore can only recognize a filename, not filter based on an internal YAML field.
2. Invisible quality status: whether sim passes, whether optimization passes, whether the literature sourcing is complete had no unified marking convention. Two suffixes, `_noref` (missing literature) and `_mw` (a pure component), were used previously, an incomplete, asymmetric convention.
3. Low test_batch efficiency: running batch against every file each time re-tests already-passing models, slowing down the validation cycle.

---

## Decision

### Three filename suffixes, everything else deprecated

| Suffix | Meaning | gitignore |
|------|------|-----------|
| `_nosim` | sim cannot run (a YAML parse error, a variable/formula reference error) | Yes |
| `_noopt` | sim passes, optimization fails (an algorithm error, a constraint violation, etc.) | Yes |
| `_noref` | missing literature source (a `TODO:SOURCE` present in `variables`/`evidence`/`formulas`) | Yes |

Combinations: `_nosim_noopt` is the conservative default state for a newly created or untested model; `_noref` can combine with another suffix (such as `_noref_nosim`).

No suffix means confirmed passing: a file satisfying all three conditions carries no suffix at all, and is in a publishable state.

### Deprecated old suffixes

| Deprecated suffix | Replacement |
|---------|---------|
| `_mw` (a pure component, with no sim/opt by design) | Remove the suffix; add `_nosim_noopt` if untested |
| `_TODO` (a draft/WIP) | Remove the suffix; add `_nosim_noopt` to mark it untested |

### gitignore

```
models/**/*_nosim*.yaml
models/**/*_noopt*.yaml
models/**/*_noref*.yaml
```

(models is an independent repo, with `.gitignore` maintained at that repo's root)

### An inverted filter for test_batch

A new `FILTER_BROKEN=true` mode tests only files whose name contains `_nosim` or `_noopt`:

```bash
FILTER_BROKEN=true MODEL_FOLDER=models/references bash script/test_batch.sh
```

This turns test_batch from "a full regression run" into "a fix queue," processing only the problem files and removing the suffix once one passes.

### reviewed: true (an optional YAML field)

A manually confirmed model can add `reviewed: true` under `metadata`, meaning the modeler has confirmed the mechanism is sound and the parameter magnitudes are correct. This is a positive signal, not a publication gate, and does not participate in gitignore.

---

## Initial Rollout (2026-06-06)

- test_batch was run against `models/papers/` (17 files) and `models/test/` (24 files); results were marked accordingly: 6 `_nosim`, 6 `_noopt` (papers), 1 `_nosim` (test).
- test_batch was run against part of `models/references/` and part of `models/scenarios/`; known-failing files were marked, and every untested file received `_nosim_noopt`.
- Every `_mw` file (10) and `_TODO` file (21) was renamed, with the old suffixes fully cleared.
- Final state: 85 `_nosim_noopt`, 5 `_nosim`, 4 `_noopt`, 8 `_noref`, 43 clean with no suffix.

---

## Related

- `docs/model.md` - the filename quality-marker section (for modelers)
- `script/test_batch.sh` - the `FILTER_BROKEN` mode's implementation
- ADR 0091 - the CLI batch-testing tool (test_batch's original design)
