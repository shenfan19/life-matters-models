# ADR 0141 - Restoring the method Field in description, Plus Moving Debug History into a history/ Directory

## Status

Implemented

## Date

2026-08-05

## Background

ADR 0097 narrowed `papers/` model `description` to three fields, `problem/result/limitations`, listing `method` among the removed fields, on the grounds that `method`'s content at the time relied heavily on framework-internal notation (T1/T2/T3/T4, NSGA-II, etc.) that a lay reader would find hard to follow.

As the number of models grew, two new problems surfaced:

1. **With `method` removed, there was no place in `description` for "which mechanisms this model combines"**: LM's core methodology is coupling several independent mechanisms rather than stacking them, which is exactly what each model's description most needs to state clearly, but the three-field structure had no dedicated place to carry it, so this information either crowded into `problem` or was left out entirely.
2. **Repeatedly debugging the same model let process traces accumulate in `problem`/`result` and `optimization.results`**: a real example is `s1/ckd_protein/ckd_protein_opt_joint_largepop.yaml`, whose `problem` field read "the current search scale may miss the tail of the front and needs to be verified at a larger scale," which is the debugging process's own self-narration, not the scientific question the model answers; its `optimization.results` had accumulated four historical run records in full, `rerun_2026-07-10`, `rerun_2026-07-11_mc_fixed`, `rerun_2026-07-14`, and `historical_stale_from_2026-06-15`, with only the last being the currently valid result and the other three having no external value beyond tracing the debugging trail, yet letting the file keep growing. This content violates the "Deliverables for the Final Reader Carry No Process Trace" section of the global `~/.claude/CLAUDE.md`, but that rule had not previously made explicit that it covers a model YAML's description and optimization results block.

## Decision

### 1. `papers/` model description restored to four fields: `problem / method / result / limitations`

- `problem`: unchanged, merging the former `brief` plus `need`, but now additionally and explicitly excludes the debugging or optimization process itself (whether the search scale is large enough, an earlier statement turning out on review to be coincidental, etc.), since that content belongs in `metadata.todo`/`metadata.log`/`history/`, not in `description`.
- **`method` (restored)**: states which mechanisms or dynamic models the model combines and which decision variable or resource-competition pathway they share, reflecting coupling rather than stacking. The field has no fixed length; when there are many mechanisms, list them one by one.
- `result`: unchanged, merging the former `result` plus `conclusion`, likewise excluding the debugging-process narrative and stating only the final simulation or optimization result.
- `limitations`: unchanged.

`method` is also restored as a formal field definition in the general schema (the nine-field set from ADR 0065), with its wording changed from "the model's structure, time step, core states, and inputs" to "which mechanisms or dynamic models the model combines and which decision variable or resource-competition pathway they share," aligning it with `papers/`'s usage; non-`papers/` models such as `references/` are not required to include this field, but are encouraged to write it when a new model is created.

### 2. Debug history versions move into a same-directory `history/` subfolder

When a model is debugged repeatedly, a superseded full version (old `optimization.results`, a rejected old `description` statement) no longer accumulates and stays inside the main YAML file. The approach: before making a change, copy the current file as-is to `history/YYYY-MM-DD_original-filename.yaml`, then keep only a clean version reflecting the current state in the main file. Content under `history/` is not bound by the "carries no process trace" rule and may honestly retain debugging detail, for the author's own later review only; the entire directory is excluded via the `.gitignore` rule `**/history/` and is not published with the repository.

`metadata.log` is unaffected and stays in the main file; it is only a one-line index of what changed and why, small in size and useful to an external reader for tracing design motivation, and does not count as a process trace that needs moving to `history/`.

## Impact

- `model.md` has been updated accordingly: the `metadata.description` section restores the `method` definition and adds a note excluding process language from `problem`; a new "Debug History Versioning: the `history/` Directory" section has been added.
- `models/.gitignore` has gained a `**/history/` entry.
- The "Deliverables for the Final Reader Carry No Process Trace" section of the global `~/.claude/CLAUDE.md` has been expanded to state explicitly that this rule covers a model YAML's description field.
- Every model file under `models/papers/` has been updated in bulk under the new rule: process language cleaned out of `problem`/`result`, a `method` field added, and historical debugging records moved into each model's own `history/`.

## Non-Goals

- This does not require `references/` models to adopt the four-field structure or to backfill `method` immediately.
- This does not turn the four fields into a hard schema constraint (the validator remains permissive).
- The existing rules for `metadata.todo`/`metadata.log` are unchanged; this ADR only adds `history/` as a new storage location and does not change what `todo`/`log` themselves are for or how they are formatted.
