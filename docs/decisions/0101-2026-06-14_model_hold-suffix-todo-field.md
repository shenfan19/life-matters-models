# 0101 - Unifying the Filename Quality Marker into `_HOLD` Plus a `metadata.todo` Task List

**Date**: 2026-06-14
**Status**: Filename convention partly superseded by [0120](0120-2026-06-23_model_drop-hold-filename-suffix.md) (the `_HOLD` suffix abolished; the `metadata.todo` field definition remains valid and is still the current convention)
**Category**: Model library management / engineering convention

---

## Background

ADR 0096 used three filename suffixes, `_nosim` / `_noopt` / `_noref`, to mark an incomplete state along the sim, optimization, and literature-sourcing dimensions respectively, with combined forms (such as `_nosim_noopt_noref`) exposing two problems in practice:

1. A suffix carries no diagnostic information: a `_noopt` file only states that "optimization failed," not why. Every review pass (whether by an AI or a human) had to rerun `--opt` and re-analyze the log and Pareto front from scratch; the diagnostic process was not reusable.
2. A binary state cannot express "runs, but is questionable": running the new CLI against the fix queue for `models/papers/` found that some models had both `--sim` and `--opt` return success (a technical PASS), but the result itself was problematic, for instance a Pareto front collapsed to a single point, or a hard constraint that even the model's own "optimal solution" example could not satisfy (an empty joint feasible region). The three-suffix system had no way to mark this "runs successfully but the result is unusable" situation, leaving it to be tracked by memory or a separate document.

---

## Decision

### 1. Unify the three status suffixes into a single `_HOLD` suffix

- `_HOLD` means this file has one or more pending items, with detail recorded in `metadata.todo` (below).
- gitignore reuses the existing rule `**/*_HOLD.yaml` (this rule already existed in `.gitignore` but had not been documented in `docs/model.md` or an ADR).
- No `_HOLD` suffix and no `metadata.todo` means confirmed passing and publishable (consistent with ADR 0096's "no suffix means passing" semantics, only the signal changes from "none of the three suffixes present" to "`_HOLD` absent plus todo empty").

### 2. Add `metadata.todo`: a structured task list

```yaml
metadata:
  todo:
    - type: nosim | noopt | noref | quality | other
      issue: "A one-sentence description of the problem"
      evidence: "Diagnostic basis: specific values/observations/reproduction steps, so the next pass doesn't have to re-diagnose"
      next: "A suggested next step, or an option left for human judgment; do not pre-decide on the human's behalf"
```

- `type` carries forward ADR 0096's three categories (`nosim`/`noopt`/`noref`), and adds:
  - `quality`: both sim and opt succeed, but the result is questionable (a degenerate front, an empty feasible region, etc.)
  - `other`: a pending item not covered by the above
- `evidence` is this change's core value: writing down the diagnostic process's conclusion (specific values, run conditions, reproduction path) so an AI or a human can pick it up directly next time and continue, without rerunning or re-deriving it.
- `next` offers options rather than a fix to execute directly; content involving a modeling judgment, a parameter's order of magnitude, a constraint threshold, is left for a human to confirm.

### 3. State transition

- Once every `metadata.todo` item is resolved and the field is emptied (removed), the `_HOLD` suffix is dropped and the file returns to a clean state.
- Partway through, some `todo` items can be kept while resolved ones are removed; as long as `todo` is non-empty, the filename keeps `_HOLD`.

### 4. Relationship to the old convention (ADR 0096)

- The three suffixes `_nosim` / `_noopt` / `_noref` and their gitignore rules are deprecated, fully migrated to `_HOLD` plus `metadata.todo` (the full migration completed 2026-06-14).
- A newly found problem is recorded uniformly using `_HOLD` plus `metadata.todo`.

### 5. Relationship to a directory-level `_HOLD`

`models/papers/` already has directory-level `_HOLD` markers such as `s3_HOLD/`, `s4_HOLD/` (a paper-structure "deferred" signal, with `**/*_HOLD/` in `.gitignore` keeping the whole directory from publishing). This is a paper-structure-dimension marker, an independent dimension from this ADR's technical/quality-dimension marker (a filename `_HOLD` plus `metadata.todo`); the two can coexist (for example `s3_HOLD/foo_HOLD.yaml`) without affecting each other.

---

## Pilot (2026-06-14)

The new convention was applied to a `_noopt` file under `models/papers/`: renamed to `_HOLD.yaml`, with `metadata.todo` added recording the diagnostic finding of "an empty joint feasible region" found while running `--opt` with the new `sim_cli` (a hard constraint mismatched with the current dynamics, where the model's own example solution drives the constrained state variable outside the constraint's range within the simulation window).

## Full Migration (2026-06-14)

The remaining roughly 100 `_nosim`/`_noopt`/`_noref` (including combined) files under `models/` have been migrated in bulk to `_HOLD` plus `metadata.todo`. The migration only performed the structural conversion from suffix to `todo`, with the `evidence` field noted as "migrated from the old suffix marker, not yet re-run through `--sim`/`--opt` diagnosis"; the fix queue in `sim_cli/batch.py` will later run `--sim`/`--opt` on each one to add the specific diagnostic evidence and clear `todo`.

---

## Related

- `docs/model.md` - the filename quality marker section
- ADR 0096 - the three-suffix convention (superseded by this ADR)
- The internal task record `2026-06-14_task_hold-todo-migration.md` - the full migration record
