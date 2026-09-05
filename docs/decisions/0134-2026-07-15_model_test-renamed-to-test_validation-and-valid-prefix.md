# 0134 - Renaming models/test/ to models/test_validation/, Adding a test_valid_ Prefix Under valid/

**Date**: 2026-07-15
**Status**: Accepted

---

## Background

The repository had two similarly named but differently purposed test directories: `models/test/` (a set of YAML fixtures plus the `cli/batch.py` batch-run approach, validating whether a model's/simulation's result is correct) and the pytest suite in the life-matters-reference-engine repository, just renamed from `tests/` to `test_verify/` (validating whether the engine's code is written correctly; see the corresponding commit in the life-matters-reference-engine repository the same day). Both were called "test," and in actual use (running `python cli/batch.py --input-dir test` over every model versus reading pytest code) it was hard to tell which was which; the request was to rename both to be self-explanatory, the pytest suite renamed after "verify," and this directory renamed after "validate," which is this decision.

## Decision

### 1. Rename the top-level directory `models/test/` to `models/test_validation/`

Symmetric with life-matters-reference-engine's `tests/` to `test_verify/`: `test_validation` corresponds to "validating the credibility of a model's/simulation's result," while `test_verify` corresponds to "verifying the engine's code is correct." See the new `models/test_validation/README.md`'s opening, which specifically explains this distinction (per the user's request to "first explain the difference between validation and verify").

### 2. Add a `test_valid_` prefix to the 35 files under `valid/`; `invalid/` unchanged

`valid/test_X.yaml` becomes `valid/test_valid_X.yaml` (for example `test_banister_v1_analytical.yaml` becomes `test_valid_banister_v1_analytical.yaml`). The 11 files under `invalid/` were already in the `test_invalid_*.yaml` format (named that way as of ADR 0125) and do not need changing. The subdirectories themselves are still called `valid/` and `invalid/`, unaffected by this rename; only the filenames inside them gain a uniform prefix, with `metadata.name` updated to match the new filename (following the pre-existing convention that a filename matches its `metadata.name`).

Why both sides needed changing, not just the directory name: the filenames under `invalid/` were already `test_invalid_*.yaml` (a product of ADR 0125); if `valid/` did not follow suit and become `test_valid_*.yaml`, the two subdirectories' filenaming rules would be inconsistent, and reading the code, a bare `test_opt_t2.yaml` would give no clue which subdirectory it belonged to or whether it was even valid.

### 3. Scope of impact: a directory rename plus a file rename stack, requiring path tokens to be handled in the same batch of replacements

Every place where `imports:`, `metadata.name`, or `description` mentions another fixture's name (such as the problem description in `test_import_top.yaml` stating "imports test_import_base AND test_import_layer") needed the new filename substituted in as well, not just the directory prefix; otherwise a three-layer-coupled reference such as `imports: [test/valid/test_import_base]` (directory name plus subdirectory plus filename) would produce a dead link at an intermediate state if the directory rename and the file rename were done in two separate steps. This was handled with a single script pass replacing both the "path form" (`test/valid/<old name>` to `test_validation/valid/<new name>`) and the "bare word form" (`<old name>` to `<new name>`, for `metadata.name` and prose mentions), applied in descending order of length to avoid prefix collisions (such as `test_opt_t1_pareto` and `test_opt_t1_single` sharing the prefix `test_opt_t1`).

Files updated accordingly in the life-matters-reference-engine repository: `cli/batch.py` (the docstring plus argparse help), `docs/reference_engine/cli.md`, `docs/reference_engine/DECISIONS.md`, `README.md` (the documentation index table), the `FIXTURES_DIR` and 4 T1-T4 fixture filenames in `gui/src/components/sim_tab/optUtils.test.ts`, the 3 GUI file-tree testid strings in `gui/e2e/specs/run-simulation.spec.ts`, the `MODEL_PATH` in `reference_engine/scripts/validate_banister.py`, and every pytest case under `test_verify/` referencing a specific model_key (`test_capacity_limits.py`, `test_same_day_duration.py`, `test_schedule_runner.py`, `test_session_cleanup.py`, `test_sim_cli_consistency.py`, `errors/*.py`). The two subdirectories `test_verify/models/test_mc_distributions/` and `test_verify/models/test_plans/` were themselves also renamed to `test_verify/models/test_valid_mc_distributions/` and `test_verify/models/test_valid_plans/`, keeping them consistent with `tests/models/README.md`'s (now `test_verify/models/README.md`'s) existing convention that a directory name equals the model's `metadata.name`.

3 review annotations in `life-matters-home`'s `paper/c_paper_s1_cn.md` referencing a specific path or filename (relating to `models/test/test_report.md` and `test_banister_v1_analytical.yaml`) were updated accordingly; the annotation content itself (the numeric conclusion) is unaffected, only the path was updated.

### 4. Historical ADRs are not rewritten retroactively

The dated ADR files already present under `docs/decisions/` (this directory) and life-matters-reference-engine's `docs/reference_engine/decisions/`, and the historical lines in both sides' `decisions/README.md` index tables describing what a given past change did, are never rewritten to the old paths just because of this rename; they record the state that was true at the time, and editing them after the fact would falsify the historical record. Only a living document that clearly represents the current state (such as `DECISIONS.md`'s top-level overview or `README.md`'s documentation index) is updated to the new paths.

## Result

- Rename: `models/test/` to `models/test_validation/`
- Rename: the 35 `.yaml` files under `models/test_validation/valid/` gain a `test_valid_` prefix, with `metadata.name` updated to match and every mutual mention in `imports:`/description text updated accordingly
- Added/rewritten: `models/test_validation/README.md` (a top-level overview, opening with an explanation of validation versus verify)
- Updated accordingly: path references in about 15 files in the life-matters-reference-engine repository (see above), and 2 subdirectories renamed under `test_verify/models/`
- Updated accordingly: path references in 3 review annotations in life-matters-home's `paper/c_paper_s1_cn.md`
- Validated: after the rename, `pytest` (`test_verify/`, 38 test cases) passes in full

## Open Questions

- The substantive content of `models/test_validation/test_plan.md` and `test_report.md`, beyond the path tokens, was not re-reviewed this time; only path substitution was done.
