# ADR 0075 - Removing the Top-Level YAML type and standalone Fields

**Date**: 2026-05-17
**Status**: Adopted
**Scope**: `models/**/*.yaml`, `docs/model_design.md`, `sim_engine`, `sim_gui`

---

## Background

The YAML schema historically had two top-level fields:

- `type: model | story`: distinguishing a "mathematical component" from a "combined analysis case"
- `standalone: true | false`: marking whether a model can run independently

## Problems

### The `type` field

A code review found this field had no behavioral effect at all:

- The `sim_engine` loader, validator, and simulator never read the `type` field.
- `api_server.py` only reads it to return as metadata to the frontend.
- The frontend only renders it as a blue tag, with no branching logic.
- 64 actual YAML files had a `type:` field, with zero impact on the run result.

### The `standalone` field

- Files that actually wrote `standalone: false` under `models/`: 0.
- All 73 "library component" models under `references/` had a complete `simulation:` block and could run independently.
- The UI code had a warning branch for this, but it was never triggered.

### At the design level

Whether something "can run independently" is essentially an engineering question, not a semantic-label question: a model that only models glucose absorption can still run without an insulin feedback loop, and its run result has value for component-level validation (parameter isolation, checking the absorption curve's shape), even when it is not clinically meaningful. Forcibly using a field to block a run instead gets in the way of debugging.

The distinction between a "component" and a "complete case" is already naturally expressed by folder location: `references/` holds reusable building blocks, `published/` holds fully validated cases, with no need for a redundant extra field.

## Decision

Remove both top-level fields, `type` and `standalone`, including:

- The schema description in `model_design.md`
- The `type:` line in every YAML file (64 of them)
- The code collecting `model_type` in `api_server.py`
- The `standalone` warning logic and the `model_type` tag in `Simulator.tsx`
- The `model_type` tag display in `Loader.tsx`
- The `model_type` field on the `DataNode` interface in `types.ts`

## Impact

| Aspect | Before | After |
|------|--------|--------|
| Model files | 64 had a `type:` line | No such field, cleaner YAML |
| "Component" indicator | UI showed a blue tag (with no real constraint) | No tag; distinguished by folder location |
| Run constraint | None (`standalone` never actually blocked a run) | None (unchanged) |
| Folder semantics | `references/` implied "library component" | Unchanged, remains the only distinguishing means |

## Out of Scope

- Changing the engine's actual logic for deciding whether something "can run" (the engine always allows any valid YAML to run).
- Changing the `type:` sub-field on each variable under `variables` (`input`/`state`/`parameter`/`evidence`, which have clear behavioral differences and are unaffected).
