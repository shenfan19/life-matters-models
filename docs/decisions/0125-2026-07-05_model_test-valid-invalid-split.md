# 0125 - Splitting models/test/ into valid/ + invalid/: Adding Error-Detection Fixtures

**Date**: 2026-07-05
**Status**: Accepted

---

## Background

`models/test/` originally held only structurally valid engine-feature examples (31 files covering import combinations, MC distributions, each K×4 optimization tier, etc.), validating that "the engine correctly loads and runs a valid model." But a review of error detection found the engine had no test coverage at all for a structurally broken model, a circular import, an evidence name conflict, a formula referencing an undeclared variable, and so on: there was neither a deliberately broken fixture nor a test asserting that loading such a model must fail, and fail with a specific reported reason (see reference 0124 in the life-matters-reference-engine repository; that same review also found that `LoaderEngine.fetch()` swallows the specific error message, logging it only).

## Decision

### 1. Split into two subdirectories by "structurally valid" versus "deliberately broken," rather than mixing them together or creating a new top-level directory

- `models/test/valid/`: the original 31 files, migrated as-is, purpose unchanged (functional examples, not real scenarios).
- `models/test/invalid/`: 11 new fixtures added, each deliberately breaking exactly one thing (`simulation.step_size`, `optimization.method`, a formula referencing an undeclared variable, the deprecated symbol `dt`, a circular import, an import reaching outside the models root, an evidence name conflict, evidence missing `baseline_ref`, a top-level YAML that is not a mapping, `end_date` earlier than `start_date`), covering four categories of checks across `validator.py`/`loader.py`/`validation.py`.

Placed under `test/` rather than a new top-level directory: neither kind is a real scenario, and both are engine-test infrastructure, consistent with `test/`'s existing purpose, just filling in its "negative" half; a new top-level directory would create a second top-level classification (alongside `papers/`/`scenarios`/`references/`/`test/`), adding cognitive load.

Each invalid fixture breaks exactly one thing: this shares its rationale with `tests/models/README.md`'s isolation principle of one variable per folder; when changing the logic of a given validation branch, it is enough to check whether the one corresponding file still fails as expected, with no worry that multiple errors in one fixture could interfere with each other and mask a regression.

### 2. Scope of impact: 3 internal `imports:` paths plus 5 external references need updating

The internal `imports: [test/test_import_base]` in `test_import_layer.yaml`/`test_import_top.yaml`/`test_step_cross_import_top.yaml` changes to `test/valid/test_import_base` (learning from ADR 0120's lesson that a rename breaks other files' `imports:` paths and must be checked one by one, not just moved on the filesystem).

5 places in the life-matters-reference-engine repository referencing the old path are updated accordingly: the docstring in `cli/batch.py`, `docs/reference_engine/cli.md`, the `FIXTURES_DIR` in `gui/src/components/sim_tab/optUtils.test.ts`, `gui/e2e/specs/run-simulation.spec.ts`, and `test/test_opt_t1_single` to `test/valid/test_opt_t1_single` in `tests/test_sim_cli_consistency.py`.

### 3. Known side effect: a whole-library or `--input-dir test` batch run reports the invalid fixtures as FAIL

`cli/batch.py` currently has no directory-exclusion mechanism (`rglob('*.yaml')` applies no filter), so scanning the entire model library or `--input-dir test` (without pointing specifically at the `valid` subdirectory) runs `--sim`/`--opt` over the models under `invalid/` too, and they are expected to FAIL, which is by design (a fixture is supposed to fail). A comment has been added in `cli/batch.py` and `docs/reference_engine/cli.md` explaining this, but no exclusion mechanism has been added to `batch.py`, and a full-library batch run has not actually been executed to confirm these FAIL lines render normally and that an extreme structure such as `test_invalid_yaml_not_dict.yaml` (a top-level list) does not crash `batch.py` itself instead of going through the normal FAIL-recording path. This is left as an open item and does not block this decision.

## Result

- Added: `models/test/invalid/` (11 YAML files plus README.md)
- Migrated: `models/test/*.yaml` to `models/test/valid/` (31 files plus README.md, with 3 internal import paths updated accordingly)
- Added: `models/test/README.md` (a top-level overview explaining the two subdirectories' purposes)
- Updated accordingly: 5 external path references in the life-matters-reference-engine repository (see above)
- Regression lock-in: `tests/errors/` (in the life-matters-reference-engine repository, 11 pytest cases, see reference 0124)

## Open Questions

- Whether `cli/batch.py` needs a directory-exclusion mechanism to keep expected FAILs out of a whole-library or broad `--input-dir` batch report: undecided, needs separate evaluation.
- The change to "clicking `test` then `test/valid` as two folder levels" in `gui/e2e/specs/run-simulation.spec.ts` is inferred only from reading `SimModelTree.tsx`'s code, without having actually launched the GUI to verify it.
