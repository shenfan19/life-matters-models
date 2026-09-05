# ADR 0152 - Renaming the optimizer: Top-Level Field to optimization:

## Status

Implemented

## Date

2026-08-25

## Background

LM format's four core structural blocks read together as VESO, corresponding to `variables`, `equations`, `simulation`, `optimizer`. ADR 0144 (2026-08-08, renaming `formulas:` to `equations:`) had already formally set VESO's mnemonic as Variables, Equations, Simulation, **Optimization**, but at the time only the mnemonic's wording was changed to match, without also renaming the fourth field itself from `optimizer:` to `optimization:`; `simulation` is an activity/process noun, while `optimizer` is an agent noun, so among VESO's four letters, only O was formed differently from the other three.

This inconsistency is not just a matter of writing style: the `simulation:` block holds a single simulation experiment's or scenario's configuration, not the "simulator" tool itself, which is exactly why `simulator` was deliberately avoided when this field was first named (`self.simulator` has remained as an internal engine attribute name, but the externally facing YAML field has long been `simulation:`). The `optimizer:` block likewise holds a single optimization experiment's configuration and results (`optimizer.results`), not the "optimizer" tool itself, and the same naming logic should apply, meaning it should have been named `optimization:` all along.

Every entry in `LM_format_1.0.md`'s v1.0 version history is marked Draft, and the specification has never been frozen and released externally; the S1 paper is this project's first formal release point facing external readers. On this basis, this rename does not constitute a breaking change to an already-published contract and does not trigger a major version bump; `LM_format_1.0.md` stays at v1.0.

## Decision

### 1. Rename the `optimizer:` top-level field to `optimization:`

The top-level `optimizer:` key is changed to `optimization:` in every LM file (175 files total under `models/` across the `life-matters-models` and `life-matters-game` repositories); sub-field paths (`optimizer.results` to `optimization.results`, `optimizer.startpoint.regimens` to `optimization.startpoint.regimens`, `optimizer.mc` to `optimization.mc`, `optimizer.algorithm` to `optimization.algorithm`, etc.) change to `optimization.*` automatically along with the top-level key rename, with no separate handling needed.

### 2. Only externally visible field names change; the engine's internal implementation details stay the same

The scope boundary was explicitly settled by the user: only field names in the YAML, the specification document, papers, GUI display text, and API response JSON field names change, since these are what a reader or user sees directly. The engine's internal module filenames (`optimizer_engine.py`, `optimizer_backends.py`, `optimizer_parsing.py`, `optimizer_eval.py`, `routes/optimizer.py`), class names, function names, Python attribute names (`self.optimizer`), local variable names, and API route paths (`/api/optimizer/*`) all keep `optimizer` unchanged, since these are the engine's own implementation detail that no one reading a model YAML or a paper ever sees; renaming all of these too would be a deep internal refactor that substantially raises risk and effort with no externally visible benefit, and is out of this scope.

At the code level, this means: the one string literal that reads or writes the YAML's top-level key (`data.get('optimizer')`, etc.) changes to `data.get('optimization')`, but the Python attribute name `self.optimizer` holding the parsed result does not change; wherever a user-facing error message, CLI output, GUI hint text, or i18n string refers to a field path (such as "no `optimizer:` block" or "`optimizer.startpoint.regimens` missing"), it changes to the corresponding `optimization` path, but a generic English phrase describing the algorithm or tool itself (such as "the optimizer searches...", "skip the optimizer", "an optimizer fitness") stays unchanged, since this usage describes the concept of the action of optimizing, or the optimizer as a tool, not the field name itself. The GUI's "Optimizer" tab/menu label is likewise kept, treated as the name of a tool panel, not a reference to the field name.

### 3. The four repositories involved

- **`life-matters-models`**: `LM_format_1.0.md` (the body text, the YAML skeleton code blocks, the section 6.2 heading, the terms table, the Inclusion Test's four questions, and the lines in the version history describing historical events), the six files under `docs/authoring/` (especially `regimens_and_optimizer.md`, renamed to `regimens_and_optimization.md`, including its body headings and tables), every relevant model YAML file under `models/` (including `test_fixtures/`, `test_validation/`, `references/`, `papers/`, `plan/`, covering both the top-level `optimizer:` key and any sentence mentioning `optimizer` in a prose field such as `description`/`ratings`), `test_fixtures/invalid/test_invalid_optimizer_missing_method.yaml` renamed to `test_invalid_optimization_missing_method.yaml`, the skill documents under `skill_agent/` that reference this field, `models/plan/lm_model_library_plan.md`, the body text of ADRs under `docs/decisions/` describing the current state (except historical ADR titles/filenames themselves, see below), and the descriptive text in the `docs/DECISIONS.md`/`docs/decisions/README.md` indexes.
- **`life-matters-reference-engine`**: `reference_engine/src/model_structure/loader.py` (reading the YAML top-level key), `core.py` (`export_to_yaml` serialization), `validator.py` (three user-facing validation error messages), `optimizer_engine.py`/`validation.py`/`routes/optimizer.py`/`routes/models.py`/`routes/files.py` (docstrings, error messages, the API JSON field name `"optimization": model.optimizer`), comments in `csv_export.py`/`mc_utils.py`, `cli/main.py`/`cli/batch.py`/`cli/runner.py` (CLI output text and the actual field-checking logic in `model_declares_step`), the regression tests under `test_verification/` (including one embedded YAML fixture), `docs/opt.md`/`cli.md`/`architecture.md`/`mc.md`/`design.md`/`data_flow.md`, `CHANGELOG.md`, and the descriptive text in the `docs/DECISIONS.md`/`docs/decisions/README.md` indexes.
- **`life-matters-game`**: the 20 scenario YAML files under `models/scenarios/social/`, and `game/src/components/StoryEditor.tsx`.
- **`life-matters-home`**: every paper draft under `paper/*.md` (S1-S4, including the abstract, body text, and any real field reference in a contribution list; already-closed review-comment or troubleshooting-record callouts are not rewritten retroactively), the task text under `tasks/` (including `archive/`) describing an actual field path, and the two living methodology documents `process/veso_debug_checklist.md` and `process/model_validation_workflow.md`; `agent_reports/`, `process/veso_case_archive.md`, the `outreach/` slides, and the content under `personal/` are not rewritten retroactively, for the reasons given below.

### 4. The GUI frontend

The `ModelFile.optimizer` field in `gui/src/types.ts` is renamed `optimization`; every component/hook that reads this field (`Loader.tsx`, `useBuilderState.ts`, `useFileTree.ts`, `useModelInit.ts`, `useSimulation.ts`, `usePlans.ts`, `useOptimizer.ts`, `Simulator/index.tsx`) switches to `.optimization` accordingly; in the four i18n files (`en`/`zh-CN`/`zh-TW`/`fr`), a string referring to literal YAML syntax (such as "optimizer: block" or "optimizer.inputs") is renamed accordingly, but the "Optimizer" tab/menu label itself, and generic text describing algorithm behavior (such as "time granularity the optimizer steps through"), stay unchanged, for the same reason as above.

## Boundaries of the Rewrite's Scope

The handling principle matches ADR 0144: historical records are not rewritten retroactively, but the user explicitly asked to narrow this principle's scope to genuine historical snapshots, excluding a living document still being referenced that would mislead a current reader.

Exceptions (kept as-is, not an oversight):

- **The titles and filenames of already-accepted historical ADRs under both repositories' `docs/decisions/`** (such as 0098 `optimizer-schedule-sustained-mode.md`, 0147 `optimizer-t1-value-step-grid-quantization.md`), and this ADR's own historical statement of what something used to be called before the rename: rewriting an ADR's filename, or a statement of what the old name used to be, would introduce a factual error, so these are kept as-is; but any statement in an ADR's body text describing what the current specification or code looks like has been updated to `optimization` per the user's request.
- **The `LM_format_1.0.md` section 10.1 version-history table**: a line's wording describing the historical event itself has been updated to current terminology (such as changing "optimizer schedule tiers" to "optimization schedule tiers"), but the 2026-08-08 line specifically recording the fact that "the old abbreviation `variables`/`formulas`/`simulation`/`optimizer` was formally named V.E.S.O." keeps the word `optimizer` as originally written, since this line describes what that abbreviation looked like at the time, and rewriting it would introduce a factual error.
- **`life-matters-home/agent_reports/`, `process/veso_case_archive.md`, the `outreach/` slides, and `personal/`**: not rewritten retroactively, following existing precedent (`outreach/` follows the exception already set by ADR 0144); `life-matters-home/tasks/` (including `archive/`) has been handled in this round per the user's request and is no longer part of this exception.
- **`life-matters-reference-engine`'s API route path `/api/optimizer/*`, its Python module filenames `optimizer_*.py`, and its internal attribute name `self.optimizer`**: kept unchanged per this scope's ruling; see decision item 2.

## Known Environment Limitations and Unfixed Pre-Existing Issues (Found but Out of This Change's Scope)

- The `export_to_yaml` method in `reference_engine/src/model_structure/core.py` serializes using the key `'simulator'` rather than the currently actually effective key `'simulation'`, a pre-existing inconsistency from before this rename (related to a legacy `simulator`/`simulation` naming carryover, not introduced by this `optimizer`/`optimization` change); this change only renamed `optimizer` to `optimization` under the same logic and did not fix this pre-existing issue.
- The `optimizer.inputs`/`optimization.inputs` field mentioned in the GUI i18n text `sim.setup.no_opt_vars`/`sim.opt.no_inputs` was itself already deprecated back in ADR 0088 (2026-05-28) and unified into `startpoint.regimens`; these two pieces of text have been referring to a stale field name ever since, and this change only renamed `optimizer` to `optimization` under the same logic, without fixing the separate, older problem that the field name `inputs` itself is stale.
- `life-matters-home/validation/validation_report.md` is suspected to be an old copy left behind after a report migration on 2026-07-06, and is not the same file as the models repository's officially published `models/test_validation/validation_report.md` (confirmed clean); this was not handled in this change, and whether to delete it is left to the user's judgment.

## Result

- Rename: `optimizer:` to `optimization:` (the LM format top-level field), with every sub-path changing accordingly; `test_invalid_optimizer_missing_method.yaml` to `test_invalid_optimization_missing_method.yaml`; `docs/authoring/regimens_and_optimizer.md` to `regimens_and_optimization.md`.
- Coverage: YAML, backend, frontend (including i18n), the specification document, the modeling guide, papers, and ADR body text (except historical titles/filenames) across the `life-matters-models`, `life-matters-reference-engine`, `life-matters-game`, and `life-matters-home` (`home/tasks`, including archive) repositories; the engine's internal module filenames, class names, attribute names, and API route paths are kept unchanged per this scope's ruling.
- `LM_format_1.0.md` stays at v1.0 (still an unpublished Draft, so this is not a breaking change to an already-published contract), and VESO's four letters now match the word-formation of the four top-level field names exactly: Variables, Equations, Simulation, Optimization.
