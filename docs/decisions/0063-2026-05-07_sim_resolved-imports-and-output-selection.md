# ADR 0063 - Resolved Imports and the Simulation Output-Selection Rule

## Status

Implemented

## Date

2026-05-07

## Background

A model like Paper3 reuses Paper2's variables, equations, and simulation configuration through `imports`. But the GUI used to display mainly the current YAML's raw content, so imported variables did not show up on the model page or the report page; the simulation side also only read the plain `output_variables`, and when a variable name did not exist it generated a zero-value curve, making a configuration error easy to mistake for the variable genuinely being 0.

At the same time, the old loader supported recursively looking up an import by a bare name. This mechanism was opaque and became unstable whenever two models shared the same name, and was not worth keeping.

## Decision

1. `imports` only supports explicit paths:
   - `published/paper2/foo` means starting from the `models/` root.
   - `models/published/paper2/foo` is kept as a compatible form.
   - `./foo`, `../foo` are relative to the current YAML file.
   - Bare-name recursive lookup is removed.

2. After merging imports, the loader records provenance:
   - `provenance.variables[var]` marks a variable's source YAML.
   - `provenance.formulas[formula]` marks an equation's source YAML.
   - The GUI's model page uses the resolved model to display variables, equations, and output variables, and shows their source.
   - The report page uses the resolved model's content and simulation results, but does not show import/source provenance, to keep the report free of internal assembly detail.

3. Output selection is interpreted uniformly by the backend:
   - When the current model defines neither `simulation.output_variables` nor `simulation.output_types`, it inherits the union of the imports' output selections.
   - When the current model explicitly defines either output field, the current model's definition takes priority and the imports' output fields are no longer mixed in.
   - When both `output_variables` and `output_types` are present, their union is taken.
   - When neither exists, or both are empty, all variables are output.
   - `output_types` only accepts `input`, `parameter`, and `state`.
   - An `output_variables` entry that does not exist is skipped with a warning, and no zero-value curve is generated for it.

## Impact

- When the Builder copies content directly into a model, no `imports` is needed, and behavior is still complete.
- A Published model that needs to reuse an earlier paper's model can keep a strongly located import path.
- What the GUI shows for a model matches what the sim actually runs.
- Source tracing is concentrated on the model page, and the report page stays a clean, result-facing output.
- A bad output variable surfaces as a warning, avoiding a silently generated, misleading zero-value curve.

## Follow-On

If more complete provenance is needed later, source information can be extended to every sub-field of `simulation` and `optimization`, with a dedicated import/source summary added to the GUI.

## Revision (2026-06-16, ADR 0107)

Decision 3's "inherit the union of the imports' output selection" rule has been revised: `output_variables` and `output_types` now follow the same behavior as other fields, a deep merge (a later import overrides an earlier one), with the root model overriding the imports. A union is no longer specially collected. See [ADR 0107](0107-2026-06-16_model_output-variables-import-overwrite.md) for detail.
