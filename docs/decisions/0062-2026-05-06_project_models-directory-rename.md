# 0062 - Renaming models/ Top-Level Directories and Updating File-Naming Conventions

**Date**: 2026-05-06
**Status**: Implemented
**Author**: shenfan19

---

## Background

`models/` had three content directories:

| Old name | Content |
|------|------|
| `source/` | Literature-based reference component models |
| `published/` | Complete scenarios bound to a paper (see ADR 0057) |
| `scenarios/` | Game scenarios and sim test scenarios still in development |

Two problems:
1. `source/` is easily misread as "source code" in an engineering context, unclear in meaning.
2. `scenarios/` was meant to contrast with `published/`, but did not convey an "in-progress/unpublished" workflow state.

At the same time, filenames under `published/` and `references/` were ordered `{id}_{topic}`, so files on the same topic (such as the paper2 and paper3 versions of `ckd_protein`) ended up non-adjacent in a directory listing, making them awkward to browse.

---

## Decision

### 1. Directory rename

| Old name | New name | Reason |
|------|------|------|
| `source/` | `references/` | States clearly that these are reference implementations based on external literature, consistent with the meaning of the internal YAML `reference` field |
| `published/` | Unchanged | Already settled in ADR 0057, an accurate name |
| `scenarios/` | `in_process/` | Forms a clear status contrast with `published/`, conveying "not yet published, still iterating" |

### 2. File-naming convention

`references/` files: `{topic}_{year}_{author}.yaml`
- `topic`: a topic word placed first so files of the same kind naturally cluster together
- `year`: taken from the year in `metadata.updated`
- `author`: an author abbreviation; a file with no traceable source is marked with a `noref` suffix, for easy bulk search and follow-up

Example:
```
flu_2026_noref.yaml
ckd_protein_muscle_2026_noref.yaml
digestive_system_2026_mw.yaml
```

`published/` files: `{topic}_{case_id}_{paper_id}.yaml`
- `topic`: a topic word placed first so different-paper versions of the same topic sit adjacent
- `case_id`: the case number within the paper (A1, B3, etc.)
- `paper_id`: the paper number (p1, p2, p3)

Example:
```
ckd_protein_a4_p2.yaml
ckd_protein_pareto_a4_p3.yaml    <- adjacent to the line above, making the shared topic obvious
```

---

## Trade-Offs

`in_process/` is a workflow-status word, not a content-type word (`scenarios` was a content type). The `test_*.yaml` files in the directory are stable CI test fixtures, not strictly "in progress." This ambiguity is accepted because:
- The `published/` versus `in_process/` contrast is intuitive and takes priority over a precisely accurate content description.
- If a finer distinction is needed later, it can be split further into `in_process/game/` and `in_process/test/`.

---

## Impact

- ADR 0022's structure snapshot: `models/source/` becomes `models/references/`.
- ADR 0057's structure snapshot and file-naming rule: updated to the new filenames.
- ADR 0058: path examples supplemented with `references/`, `in_process/`.
- CLAUDE.md's code-structure description updated.
- No change needed to the backend `api_server.py` or the frontend `ModelBuilder.tsx` (recursive scanning, see ADR 0058).
