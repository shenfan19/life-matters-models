# 0136 - Renaming test_verify/ to test_verification/, Renaming models/validation/ Back to models/test_validation/

**Date**: 2026-07-21
**Status**: Accepted

---

## Background

Earlier the same day (2026-07-21), ADR 0135 split the mismatched `models/test_validation/` (whose content was actually the `valid`/`invalid` fixtures used for verification) into `models/test_fixtures/` (the fixtures) plus the newly created `models/validation/` (the genuine literature-benchmarking/optimization-plausibility results), and its decision text explicitly discussed and rejected the option of renaming `test_fixtures` to `test_verification`, on the grounds that it would collide even more badly with life-matters-reference-engine's `test_verify/`.

Later that day, while actually updating `validation_report.md`'s simulation results, the user raised two separate naming questions: (1) can the newly produced validation CSVs go into `test_fixtures/`; (2) can `test_verify/` be renamed to `test_verification/`, to align with the `verification_report.md` it contains and to make the validate/verify sides' directory names more symmetric.

Question (1) is easy to answer: no, since it would break the boundary between "fixture data" and "validation results" that ADR 0135 had just established that same day; both `test_fixtures/README.md` and `validation_report.md`'s opening specifically state that the two are separate and independent, and mixing them together would erase that line.

Question (2) triggered this ADR: if `test_verify/` is renamed to `test_verification/`, the very reasoning ADR 0135 used to reject `test_fixtures` to `test_verification`, avoiding a name collision with `test_verify`, changes along with it, since the name `test_verify` would no longer exist and `test_fixtures` and `test_verification` would not collide. But the name `test_fixtures` already accurately describes its content (reusable test YAML data) and does not need changing again; that part of ADR 0135's decision is unaffected and does not need revisiting.

What genuinely needs revisiting is `models/validation/`: ADR 0135 named it with a bare name, no `test_` prefix, mainly because `test_validation` had just been rejected for being mismatched (holding fixtures rather than validation results), and reusing it right away would create confusion, "the same name now holding something entirely different," so a bare name was chosen as the next-best option. But the situation is now different: `models/validation/` currently holds only genuine validation content (`validation_report.md` plus the `reports/` CSVs; nothing else has been mixed in since ADR 0135 set it up), so the "mismatch between name and content" reason for rejecting it no longer applies. If `test_verify/` is also renamed to `test_verification/` at the same time, `models/test_validation/` (renamed back to this name) and `test_verification/` would form a clean "validation <-> verification" pairing, fitting the two repositories' long-standing convention of symmetric V&V terminology (ADR 0134's original motivation) better than the bare name `models/validation/`.

## Decision

### 1. `test_verify/` to `test_verification/` (the life-matters-reference-engine repository)

Renamed; the content of `README.md`/`verification_report.md`/`errors/`/`models/` subdirectories is unchanged, only the directory name itself changes; every file referencing this path is updated accordingly (see "Scope of Impact" below).

### 2. `models/validation/` to `models/test_validation/` (the life-matters-models repository)

This does not reopen ADR 0135's judgment about whether to keep fixture content and validation content in separate directories; that judgment (`test_fixtures/` holds only fixtures, with the genuine validation results in a directory of their own) is fully preserved, and the two kinds of content stay physically separated in two directories. This only changes the smaller question of whether the genuine-validation-results directory should carry a `test_` prefix: when ADR 0135 set it up, a bare name was chosen because the name had just been rejected once, and now that the mismatch no longer holds (this directory has held only validation content since it was created), and the `test_verify` to `test_verification` rename lets the effect ADR 0134 originally wanted, "two `test_`-prefixed directories paired symmetrically," genuinely hold for the first time, it is renamed back.

The content of `validation_report.md` and the `reports/` subdirectory is unchanged, only the parent directory name changes; every file referencing this path is updated accordingly.

## Scope of Impact

**life-matters-reference-engine repository**: `pytest.ini` (`testpaths`), `.gitignore` (a path comment), `scripts/check_hardcoded_constants.py` (`EXCLUDE_DIR_PARTS`), the self-references in `README.md`/`verification_report.md`/`errors/README.md`/`models/README.md` inside `test_verification/` and in the 3 `test_*.py` files under it, the root `README.md`, `docs/reference_engine/{DECISIONS.md,cli.md,evidence/conversion.md,impl.md,mc.md}`, the module docstrings in `reference_engine/scripts/validate_banister{,_step_grid}.py`, and a comment in `gui/e2e/specs/run-simulation.spec.ts`. The `test_verify` reference in the body of historical ADR 0056 is left unchanged, per the convention that historical records are not rewritten retroactively.

**life-matters-models repository**: `models/test_fixtures/{README.md,fixture_catalog.md,invalid/README.md}`, and the fixture YAML comments under `valid/` and `invalid/` referencing `test_verify/errors/` and `test_verify/verification_report.md` (8 `test_invalid_*.yaml` files plus the 7-file `test_valid_banister_v1_*.yaml` series plus `banister_step_convergence_grid_POINTER.yaml`); `models/test_validation/validation_report.md`'s own 4 references to `test_verify/verification_report.md`, and its 3 references to the `models/validation/reports/` CSV path (newly written in the day before this rename by a separate validation task, and needing to be updated in step with the parent directory's rename). A note has been added to the "relationship to `models/validation/` and `test_verification/`" section of `models/test_fixtures/README.md`, explaining that this directory once briefly used the name `test_validation` (before the ADR 0135 split), and this ADR, that same afternoon, renamed the newly created validation-results directory back to that same name, two different directories that happened to use the same name at different times, so a reader does not mistake it for the rename having been undone.

**life-matters-home**: the `models/validation/` and `test_verify/` path references in `process/model_validation_workflow.md`; `tasks/2026-07-21_report_validation.md` (an earlier validation task record from the same day, whose original path description is kept per the "no retroactive rewriting" principle, with a note appended at the end stating the current path has since changed). The historical archive file `tasks/archive/2026-07-17_task_docs-code-drift-engine-cli-gui.md` is left unchanged per convention.

**Historical ADR body text left unchanged**: the two index-summary lines for 0134 and 0135 in `docs/decisions/README.md` describe what those two ADRs decided at the time, and are not rewritten retroactively; only a new index line for this ADR is added.

## Result

- Rename: `test_verify/` to `test_verification/` (life-matters-reference-engine)
- Rename: `models/validation/` to `models/test_validation/` (life-matters-models)
- Updated accordingly: about 35 path references across both repositories plus life-matters-home
- ADR 0135's core judgment (physically separating fixtures from validation results into two directories) is unchanged; this only adjusts whether the validation-results directory carries a `test_` prefix

## Open Questions

- The `test_verify` reference in the body of `docs/reference_engine/decisions/0056-2026-05-04_project_three-tier-validation-framework.md` was not updated (per the convention that historical records are left unchanged); a reader clicking through from that ADR in the future may need to mentally apply the rename mapping themselves, which does not block this rename.
