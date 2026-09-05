# Model Lifecycle Fields: todo / log / history / reviewed

## Status Markers

### `metadata.todo` (ADR 0120; the filename carries no publication status)

Whether a model file is publishable depends solely on whether `metadata.todo` exists and is non-empty, independent of the filename. The old convention embedded this status in a `_HOLD` filename suffix, which ADR 0120 abolished, since renaming a file breaks other files' `imports:` paths and had caused `Cannot load model` failures.

```yaml
metadata:
  todo:
    - type: nosim | noopt | noref | quality | other
      issue: "One-sentence description of the problem"
      evidence: "Diagnostic basis: specific values/observations/reproduction steps, so the next pass does not need to re-diagnose"
      next: "A suggested next step, or an option left for human judgment; do not pre-decide on the human's behalf"
```

- `type` values: `nosim` means the sim cannot run; `noopt` means the sim succeeds but optimization fails; `noref` means a literature source is missing (`TODO:SOURCE`); `quality` means both sim and opt succeed but the result is questionable, such as a degenerate Pareto front or an empty feasible set; `other` covers everything else.
- `evidence` is the core field. Write down the specific values or observations obtained during diagnosis so the next pass, whether by an AI or a human, does not need to rerun the diagnosis.
- No `metadata.todo` (or an empty one) means confirmed and publishable: `--sim` passes, `--opt` passes (or is skipped automatically when there is no `optimization:` block), every parameter has a literature source, and the results raise no doubts.
- Once every `todo` item has been resolved, delete the field and the file returns to a clean state, with no filename change required.

### reviewed: true

An optional field, `reviewed: true`, can be added under `metadata` to indicate that the modeler has manually confirmed the mechanism is sound and the parameter magnitudes are correct. This is not a publication gate.

---

## Change History: `metadata.log` (ADR 0103)

An optional field recording a model's change history, compensating for the difficulty of tracing a single model's edit history through git log when commits bundle multiple files:

```yaml
metadata:
  log:
    "2026-06-14_10-30-00":
      change: "One-sentence description of what changed"
      reason: "Why it changed"
```

- The key is a `YYYY-MM-DD_HH-mm-ss` timestamp in local time; lexicographic order of the string matches chronological order, new entries are appended at the end, and existing entries are never edited or deleted.
- Each entry carries only two fields, `change` and `reason`. What changed can already be verified from a git diff, so the value lies in `reason`, which supplies the motivation or background a git diff cannot show. Any related ADR or `metadata.todo` item number is written directly into the `reason` text rather than given its own field.
- `metadata.log` complements `metadata.todo`: one looks forward to pending work, the other looks back at completed work. After resolving a `todo` item, an entry describing the resolution can be appended to `log` before the item is removed from `todo`.
- Only changes with semantic effect are recorded, such as adjustments to parameters, equations, or constraints, structural changes, or updates to literature backing; formatting or spelling fixes need not be recorded.
- This is an optional field and not a publication gate; existing models are not required to backfill it, and new entries can simply start from the next meaningful change.

---

## Debug History Versioning: the `history/` Directory (ADR 0141)

When a model is debugged repeatedly, for instance re-running it after multiple `optimization` configuration changes, or rewriting `description` several times to reflect diagnostic conclusions, the full content of older versions does not stay in the main YAML file. Accumulating past reruns' `optimization.results`, superseded `description` wording, and the diagnostic process itself in the same file lets it grow without bound and leaves new readers unable to tell which part reflects the current, valid conclusion. This is a separate concern from `metadata.log`: `metadata.log` keeps only a one-line index of what changed and why, stays small, and lives in the main file, whereas `history/` stores the full content of superseded files, which can be large and is not suitable for a publicly released file.

Approach: before debugging toward a new version, first copy the current file as-is into a `history/` subfolder in the same directory, with the filename prefixed by a date stamp (`history/YYYY-MM-DD_original-filename.yaml`), then return to the main file and delete the superseded content, keeping only a clean version that reflects the current state. The content under `history/` is for the author's own later review of the debugging trail; it is not bound by the "deliverables for the final reader carry no process trace" rule, so it may honestly retain debugging detail, failed attempts, and intermediate values. The entire `history/` directory is excluded from version control by the repository root `.gitignore` rule `**/history/` and is never published with the codebase, so external readers only ever see the final version of the main file.

The same treatment applies to a `metadata.todo` item that has been resolved but carries a long diagnostic narrative: once resolved, remove it from `todo` per the existing rule, and if the narrative still has reference value, move it into `history/` rather than leaving it "for now" in the main file's `todo` or `results`.

---
