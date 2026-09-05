# ADR 0107 - Unifying output_variables / output_types Import Behavior to Overwrite (Replacing Union)

## Status

Implemented

## Date

2026-06-16

## Background

ADR 0063 designed a special rule for `output_variables` and `output_types`: when the root model does not define them, take the union across all imports. Every other field (`start_date`, `end_date`, `step_size`, etc.) follows a deep merge instead (a later import overrides an earlier one, and the root model overrides all imports in the end).

This exception rule caused two problems:
1. Inconsistent behavior: model authors had to remember two separate rules.
2. A union can introduce noise: if model A imports B and C, where B's `output_variables` is `[x]` and C's is `[y]`, the result becomes `[x, y]`, but A's author may only have wanted C's `[y]` (because C is the more complete model). A model author has a clear intent for each model, and a union can instead create confusion.

## Decision

`output_variables` and `output_types` are no longer treated specially and now follow the same behavior as every other field:

- When the root model does not define them: they inherit the value from the last import (the natural result of a deep merge, where a later import overrides an earlier one).
- When the root model explicitly defines either output field: the root model's definition takes priority; if only one of the two is defined, the other field's value inherited from an import is cleared as well (to prevent the two filtering mechanisms from being accidentally mixed).
- When neither field exists anywhere, or both are empty: all variables are output.

## Implementation

- `sim_engine/src/model_structure/loader.py`: removed the `imported_output_variables` / `imported_output_types` union-collection logic and the `_append_unique` static method; `merge_dicts` now handles override inheritance naturally.
- `model.md`: updated the "output variable selection rule" section.
- ADR 0063: added a note about this revision.

## Impact

- For a model with multiple imports that each define `output_variables`, behavior changes from "the union" to "the last import's value." In practice, such models are nearly nonexistent (a model usually has only one import, or the root model defines its own output variables).
- The logic is simpler and the rule is more uniform, so model authors no longer need to remember a special case.
