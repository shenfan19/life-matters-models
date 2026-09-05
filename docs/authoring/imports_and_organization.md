# Imports and Model Organization

## Import Merging and Output Selection

`imports` only supports explicit paths:

- `papers/paper2/ckd_protein_a4_p2`: starting from the `models/` root, omitting `.yaml` and the `models/` prefix.
- `references/medical/physiology/glucose_regulation_2026_mw`: same convention, deep paths written out in full.
- `./local_component`, `../paper1/foo`: relative to the directory the current YAML file is in.

Bare-name lookup is disabled; `imports: ckd_protein_a4_p2` will not recursively search `models/`, so the full relative path must be written out.

### Merge Order and Override Rules

Merge order:

1. Imports load in list order, and later entries override earlier ones.
2. The current file always overrides every import, regardless of how the imports list is written.
3. Circular imports are rejected automatically (A to B to A is not allowed).
4. A file imported multiple times through diamond dependencies, such as A importing both B and C while both B and C import D, loads only once and is not stacked repeatedly.

Override mechanism (deep merge):

The merge algorithm is a full-field recursive deep merge. This is not limited to `simulation` and `optimization`; every top-level block, including `variables`, `equations`, `metadata`, `simulation`, and `optimization`, follows the same rule.

| Situation | Result |
|------|------|
| Both the root model and an import define the same variable or equation | The root model's version fully replaces the import's version (deep merge, with sub-fields likewise preferring root) |
| Only an import defines the variable or equation | It is kept, unaffected by the root model |
| Both the root model and an import define `simulation.start_date` | The root model's value overrides the import's value |
| An import has `simulation.plans` and the root model does not | The import's `plans` is kept |

Typical usage: a component model under `references/` usually has its own `simulation` block for standalone runs; after import, the root model's `simulation` overrides its start and end dates and step size, which is the expected behavior, since a component's simulation configuration is only meant for running the component standalone.

Output variable selection rules:

- If the root model defines neither `simulation.output_variables` nor `output_types`, it inherits the output selection from the last import, consistent with the deep-merge behavior of other fields.
- If the root model explicitly defines either output field, the root model's definition takes priority; if only one of the two is defined, the other field's value inherited from the import is cleared as well.
- If neither field exists anywhere, or both are empty, all variables are output.

When the GUI loads a model, it displays the resolved model: variables, equations, output variables, `simulation`, and `optimization` all reflect the result after merging imports. The model page marks which YAML file each field's value came from (provenance).
- `output_types` only supports `input`, `parameter`, and `state`.
- A variable named in `output_variables` that does not exist is skipped, with a warning surfaced in the API and GUI; a zero-value curve is no longer generated for it.

---

## Model Classification System

A three-level directory: `models/references/{L1}/{L2}/{L3}/file.yaml`

| L1 | L2 | Description |
|----|----|----|
| medical | physiology / nutrition / fitness / disease / medicine / surgery / psychology | Physiology and medicine |
| social | economy / conflict / law / psychology / technology / demography | Socioeconomics and sociology |
| environmental | climate | Environmental science |
| risk | actuarial | Actuarial science and risk |

The full L3 breakdown is in `docs/decisions/0022-models-three-level-taxonomy.md`.

### `references/` Directory Conventions

Models under `references/` fall into two categories with different optimization requirements:

| Type | Characteristics | Optimization requirement |
|------|------|--------------|
| Standalone reference model (fitness, disease, nutrition, etc.) | Has its own `input` variables and `simulation.plans[*].regimens`, and can run directly | Should have an `optimization` block |
| Deep physiological component (the `_mw` series under physiology/, such as `digestive_system`, `insulin_system`, `glucose_regulation`) | Has no `input` variables and is mainly a building block for `import`; running it standalone has no physiological meaning | Does not need `optimization` |

Rule of thumb: if a model's `variables` contains no variable of `type: input`, it is a pure component and does not need optimization.

---
