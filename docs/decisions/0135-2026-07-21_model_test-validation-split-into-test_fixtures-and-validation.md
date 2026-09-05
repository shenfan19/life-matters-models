# 0135 - Splitting models/test_validation/ into models/test_fixtures/ + models/validation/

**Date**: 2026-07-21
**Status**: Accepted

---

## Background

ADR 0134 (2026-07-15) renamed `models/test/` to `models/test_validation/`, motivated by symmetric naming with life-matters-reference-engine's `tests/` to `test_verify/` ("validate corresponds to a model's credibility, verify corresponds to whether the engine's code is correct"). But that directory's actual content, `valid/` (structurally valid engine-feature examples) plus `invalid/` (deliberately broken error-detection fixtures), had been, ever since ADR 0125, about whether the engine can correctly load, run, or reject a given YAML, which belongs to verification (whether the code is written correctly) in the V&V framework, not validation (whether a model represents the real world). `valid/README.md` itself had already long stated that "neither represents a real clinical or social scenario and neither needs literature-parameter calibration."

The genuine validation content, section 1 "Literature Benchmarking" of `models/test_validation/validation_report.md`, never depended on any `valid/`/`invalid/` fixture at all, pointing directly instead to real paper models such as `models/papers/s1/banister/`, `models/papers/s1/ckd_protein/`, and `models/papers/s4/hypertension_gout/`. The two had simply happened to share the same directory historically, unrelated in content. This mismatch between name and content was found during a discussion on 2026-07-21 about the relationship among `verification_report.md`, `validation_report.md`, and `test_verify/`.

## Decision

### 1. Split the top-level directory into two

- `models/test_validation/` becomes `models/test_fixtures/`: holding only the YAML fixtures used for engine verification (the `valid/` and `invalid/` subdirectories unchanged, and the `test_valid_`/`test_invalid_` filename prefixes unchanged; these "valid/invalid" are generic engineering terms, whether a YAML's structure is valid, a different matter from V&V's specialized term "validation," carrying no ambiguity on their own and not needing to change).
- A new `models/validation/` created: holding only `validation_report.md` (literature benchmarking, optimization plausibility, the API-IO boundary, and per-model scientific-content check results), plus a later `reports/` CSV archive directory. This sits alongside `papers/`, `references/`, `scenarios/`, and `test_fixtures/`, no longer subordinate to the fixture directory.

Why not call it `test_verification` directly: renaming `test_fixtures` to `test_verification` to echo the verify terminology was considered, but it would collide even more badly with life-matters-reference-engine's `test_verify/`, two nearly identical directory names, harder to tell apart than the current `test_validation` versus `test_verify` at telling "which one holds data and which one holds assertions." `test_fixtures` accurately describes its content (reusable test YAML data) and has no literal overlap with `test_verify`.

### 2. Renaming `fixture_catalog.md` (formerly `validation_catalog.md`) plus not creating a new report file

`validation_catalog.md` (a per-fixture overview of `valid`/`invalid`) migrates along with the directory and is renamed `models/test_fixtures/fixture_catalog.md`, with its title and self-references updated accordingly. No new "verification_report.md" is created for `test_fixtures/`: `test_verify/verification_report.md` (in the life-matters-reference-engine repository) already brings the pytest suite consuming these fixtures (`test_verify/errors/`) into its own report's scope in its section 1.1, so the whole project should have only this one verification-result report; `test_fixtures/` only needs a README plus a catalog for orientation, not a separate result report.

### 3. Scope of impact: about 30 path references updated across both repositories plus home

**life-matters-models repository**: self-references in `models/test_fixtures/{README.md,fixture_catalog.md,valid/README.md,invalid/README.md}`; 3 import paths (`test_valid_import_layer/top.yaml`, `test_valid_step_cross_import_top.yaml`); the import paths of 3 invalid import fixtures (`test_invalid_import_circular_a/b.yaml`, `test_invalid_import_escapes_root.yaml`); `models/validation/validation_report.md`'s own cross-references; `models/papers/s1/banister/banister_step_convergence_grid_POINTER.yaml` (incidentally fixing an earlier, pre-existing wrong reference here too, where this file had labeled the step-size convergence protocol "V4," while `verification_report.md` numbers that protocol V2, with V4 being something else entirely; see the next item).

**life-matters-reference-engine repository**: `test_verify/{README.md,verification_report.md,models/README.md,errors/README.md}`; the fixture-loading path strings in 9 pytest files under `test_verify/`; the `MODEL_PATH`/`MODEL_DIR` in `reference_engine/scripts/validate_banister{,_step_grid}.py`; `cli/batch.py`, `docs/reference_engine/{cli.md,DECISIONS.md,evidence/conversion.md}`, the root `README.md`; the `FIXTURES_DIR` in `gui/src/components/sim_tab/optUtils.test.ts`; 3 GUI file-tree testid strings in `gui/e2e/specs/run-simulation.spec.ts` (including one bare directory-name testid missed earlier and found incidentally this time). `test_verify/README.md`'s "relationship to `models/test_validation/`" section has been rewritten under the new split, previously describing `test_fixtures` (before its rename) as "the validate side," now accurately describing its verify nature, with a new, separate note added about `models/validation/`.

**life-matters-home repository**: `process/model_validation_workflow.md` (the `validation_report.md` path, the CSV archive path); the active references written that same day in `tasks/task_index.md` and `tasks/2026-07-21_task_gui-invalid-model-error-display.md`. The body text of historical ADRs (0134 and its index line in `decisions/README.md`), files already archived under `tasks/archive/`, and the review annotations in `paper/c_paper_s1_cn.md` referring to the already-stale, independent `test_plan.md`/`test_report.md` from before this change are all left unchanged, per the convention that historical records are not rewritten retroactively.

### 4. An incidental fix: the Banister step-size convergence protocol number V4 to V2

`verification_report.md`'s protocol numbers are V1 (a day-by-day comparison against the analytical solution) and V2 (step-size convergence testing); V3/V4 were later assigned to the "training adaptation-fatigue" subsection's literature-scenario-reproduction check points (the timing of the peak performance after tapering, and the timing of supercompensation) in the sister file `validation_report.md`, entirely unrelated to step-size convergence. The 6 `test_valid_banister_v1_step_*.yaml` fixtures and `banister_step_convergence_grid_POINTER.yaml` had previously referenced the "protocol V4" number from an earlier, now-nonexistent `test_plan.md`; these were updated at the same time to match `verification_report.md`'s current V2.

## Result

- Rename: `models/test_validation/` to `models/test_fixtures/` (the `valid/`/`invalid/` subdirectories and filenames unchanged)
- Rename: `models/test_validation/validation_catalog.md` to `models/test_fixtures/fixture_catalog.md`
- Added: `models/validation/`, with `validation_report.md` moved in
- Updated accordingly: about 30 path references across both repositories plus home (see above)
- Validated: `pytest test_verify/errors/ test_verify/models/ test_verify/test_sim_cli_consistency.py test_verify/test_same_day_duration.py test_verify/test_schedule_runner.py` all pass; running `reference_engine/scripts/validate_banister.py` (at its new path) confirmed it correctly finds and loads the model

## Open Questions

- The batch report from `cli/batch.py --input-dir test_fixtures` (without `/valid`) scanning the whole `test_fixtures/` was not re-run to check its formatting after this rename; the expected behavior matches before the rename (a FAIL under `invalid/` is by design), and no reason was found requiring special handling, left for a future routine check, not blocking this rename.
