# 0120 - Abolishing the `_HOLD` Filename Suffix, Determining Status Solely by `metadata.todo`

**Date**: 2026-06-23
**Status**: Implemented (partially supersedes [0101](0101-2026-06-14_model_hold-suffix-todo-field.md): the filename suffix part is abolished, while the `metadata.todo` field and its semantics are unchanged)
**Category**: Model library management / engineering convention

---

## Background

ADR 0101 used a dual signal, a filename `_HOLD` suffix plus `metadata.todo`, to indicate "this file has pending items"; the `_HOLD` suffix itself carried no information, only reflecting the fact that `metadata.todo` was non-empty, embedded into the filename.

A problem surfaced in practice: once a todo item was resolved and the `_HOLD` suffix removed, the file's path changed, and every `imports:` path referencing that file needed updating accordingly, or `--sim`/`--opt` would report `Cannot load model`. This is not a hypothetical risk; both `papers/s4/ckd_protein_pareto.yaml` and `papers/s4/hypertension_gout_3obj.yaml` had this exact failure (an imports path not updated when the upstream file was renamed).

## Decision

### 1. The sole signal for status: `metadata.todo`

- `metadata.todo` present and non-empty means the file has a pending item (a draft/unconfirmed state).
- `metadata.todo` absent or empty means confirmed passing and publishable.
- The filename is completely decoupled from publication status: a draft no longer requires any filename suffix, and once todo is emptied, the file does not need to be renamed.

### 2. What is abolished

- The `_HOLD` filename-suffix convention (the filename part of ADR 0101) is abolished.
- The `**/*_HOLD.yaml` and `**/*_HOLD/` rules in `.gitignore` are removed; the publication gate now relies on an external script reading the `metadata.todo` field, no longer on a filename pattern.
- The structure of `metadata.todo` and the semantics of its `type`/`issue`/`evidence`/`next` sub-fields are unchanged, carrying forward ADR 0101's original definition.

### 3. Relationship to a directory-level `_HOLD` (the paper-structure dimension)

The directory-level `_HOLD` mentioned in ADR 0101 section 5 (such as `papers/s3_HOLD/`, indicating a paper structure is "deferred") has no instance in the current repository; this ADR also removes that convention's documentation description. Should a future signal for "defer publishing a given directory" be needed, it should use a different mechanism that does not depend on a filename or directory name (such as a `.draft` marker file placed in the directory, or an explicit declaration in that directory's index file), to avoid repeating the same class of problem ADR 0101 exposed.

## Migration (2026-06-23)

104 `*_HOLD.yaml` files under the internal development working copy had their suffix removed; after renaming, it was confirmed that zero `imports:` references and zero cross-file prose mentions pointed to these files' old names, a pure filename change requiring no corresponding update to any other file.

## Related

- ADR 0101 - the original convention (its filename part is superseded by this ADR; the `metadata.todo` field definition remains valid)
- `docs/model.md` - the filename quality-marker section
