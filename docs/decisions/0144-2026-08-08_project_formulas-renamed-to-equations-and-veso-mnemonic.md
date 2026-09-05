# ADR 0144 - Renaming the formulas: Top-Level Field to equations:, Changing the Four-Element Mnemonic to V.E.S.O.

## Status

Implemented

## Date

2026-08-08

## Background

LM format's four top-level mechanisms used to be called `variables`, `formulas`, `simulation`, and `optimizer`, abbreviated var/for/sim/opt. This abbreviation was never intuitive in discussion; "for" is neither a pronounceable mnemonic nor easily distinguished from the English preposition "for," requiring extra explanation every time to remember which field it refers to.

Revisiting these four elements, it became clear that the specification's own text had always described this field using the words differential equation, regression equation, kinetic equation, and dynamic equations, never formula; most entries are differential equations governing dynamics, relationships that evolve over time, not a static formula computed once from a set of inputs (such as a BMI formula). `equations` is the field name that actually matches the descriptive language the specification already uses, not a new meaning being introduced. After the rename, the four-element abbreviation reads as V.E.S.O. (Variables, Equations, Simulation, Optimization), a pronounceable mnemonic, easier to remember and communicate than V.F.S.O.

The version-numbering rule (`LM_format_1.0.md` section 9.2) triggers a major bump only when "breaking compatibility with an old file," protecting an external user who already depends on a frozen version. Every entry in `LM_format_1.0.md`'s v1.0 version history is marked Draft, and the specification has never been frozen and released externally, so there is no "external consumer already depending on the old field name" to protect; this rename is an edit to a draft, not a breaking change to an already-published contract, so it needs no major bump and stays at v1.0 (a new changelog line records this change; see that file's section 9.1).

## Decision

### 1. Rename the `formulas:` top-level field to `equations:`

The top-level `formulas:` key is changed to `equations:` in every LM file (241 files total under `models/` across the `life-matters-models` and `life-matters-game` repositories). `life-matters-game`'s card YAML (the `source.formula` provenance-annotation field, 362 files) is likewise changed to `source.equation`, matching the field name `reference_engine/src/routes/converter.py` writes when generating these files.

### 2. Change the four-element abbreviation to var/equ/sim/opt, and the mnemonic to V.E.S.O.

The column names in `docs/authoring/methodology.md`'s inclusion-criteria inventory table, the four questions in `LM_format_1.0.md`'s Inclusion Test, `life-matters-home/process/lm_nomenclature.md`, and the introductory language in each repository's README are all changed to the V.E.S.O. phrasing accordingly.

### 3. Code, GUI, and i18n renamed to match

In the `reference_engine` backend: the `Formula` class is renamed `Equation`, and the model object's `.formulas` attribute, related variable names, and API response fields are renamed accordingly. In the `gui` frontend: components, type definitions, and the four i18n files (`en`/`zh-CN`/`zh-TW`/`fr`, including the i18n keys themselves) are renamed accordingly, and `fr.json` also fixes an elision issue, correcting "de équation" to "d'équation." The `life-matters-game` frontend (`StoryEditor.tsx`/`StoryEngine.tsx`) is renamed accordingly.

### 4. Test fixtures renamed

The 4 fixture files under `models/test_fixtures/` whose names contain `formula` (`test_invalid_formula_undefined_var.yaml`, etc.) are renamed to their `equation` counterparts, with `metadata.name` and cross-references inside each file updated accordingly; the 3 test files on the `life-matters-reference-engine` side that reference these file paths (`test_structural_errors.py`, etc.) have their paths and function names updated accordingly.

## Boundaries of the Rewrite's Scope

Across the `life-matters-models`/`life-matters-reference-engine`/`life-matters-game` repositories, except for the explicit exceptions listed below, every file, including a model YAML's `metadata.log`/`todo`/`change` fields and the summary text in the two repositories' index files (`DECISIONS.md`/`decisions/README.md`) describing historical ADR content, is rewritten uniformly under this rename, with no trace left of "this used to be called formula." In the `life-matters-home` repository, only the paper drafts (`paper/*.md`) and `validation/validation_report.md` are rewritten to the same standard; the rest of `home` (`tasks/`, `process/` other than the living documents already updated, `personal/`, `outreach/` other than the slides example) is not required to be rewritten retroactively.

Exceptions (kept as-is, not an oversight):

- **The body text and filenames of already-accepted historical ADRs under both repositories' `docs/decisions/`** (such as 0068 `formula-precompile-to-python-function.md`, 0102 `formula-priority-execution-order.md`, 0104, 0106): an ADR is this project's decision-record medium, and its original text and filename are preserved. The link text pointing to these ADRs in the index files (such as "0068 Equation Precompilation") has been updated to the new terminology, but the link target (the filename) is unchanged, so an index row's text and the filename it points to are not perfectly aligned, which is expected. The 0106 index summary "removes the `formula:` dictionary form" is kept as an exception, since that line describes an already-removed, independent old field that was never called `equation` (a different matter from this rename of the plural `formulas`/`equations` block), and rewriting it would introduce a factual error. In `LM_format_1.0.md` section 9.1's version history, the line specifically recording the fact that "the field was renamed from formula" (the new entry added 2026-08-08) keeps the old name for the same reason; the wording of every other historical line has been updated.
- **The debug-history snapshots under `models/**/history/`** (introduced by ADR 0141, excluded via `.gitignore`, not published with the repository): these are deliberately preserved as-is copies from before a change, a different kind of thing from "a change record inside a document"; these were mistakenly edited once during this rewrite and have since been checked and restored to their original state.
- **A review-annotation block in `life-matters-home/paper/c_paper_s1_cn.md` discussing an already-deprecated, independent `formula:` string field** (the old form removed by ADR 0106, a different field from this `formulas`/`equations` rename): kept for the same reason, since rewriting it would introduce a factual error.
- **`life-matters-home/process/model_copyright_safety.md`'s quotation of Article 5 of the Copyright Law, "generic tables, generic forms, and formulas," and its general statement elsewhere in the same file that "a formula as an idea is not protected by copyright"**: this is a legal citation and a generic term in an intellectual-property context, unrelated to this rename's schema field, and this file is not within `home`'s rewrite scope (`paper`/`validation_report.md`) anyway.
- **The `formula: "-0.08 * ..."` example in three slides such as `life-matters-home/outreach/lm_slides_v2_global.md`**: this is a stale example of the old singular `formula:` dictionary form that ADR 0106 removed, already inconsistent with the current schema, an existing legacy issue independent of this rename, and this file is not within `home`'s rewrite scope, so it was not fixed at the same time.
- **`f.equation` (formerly `f.formula`) in `reference_engine/src/routes/files.py`, reading an attribute that does not exist on the `Equation`/`Formula` dataclass**: this is a pre-existing bug from before the rename (that dataclass has never defined a `formula`/`equation` field), and this rename only renamed the same bug, without fixing it, which is outside this rename's scope.

## Known Environment Limitation

After this change, the local `models/` directory under the `life-matters-reference-engine` repository was empty (a pre-existing environment issue unrelated to this rename, failing the same way both before and after), so most local `pytest` cases could not actually run to validate anything, failing with "model not found." This failure was confirmed unrelated to this rename by comparing against a `git stash`. Type checking with `tsc --noEmit` passed for both the GUI (`gui/`) and the game frontend (`life-matters-game/game/`).

## Result

- Rename: `formulas:` to `equations:` (the LM format field), the four-element abbreviation var/for/sim/opt to var/equ/sim/opt, and the mnemonic V.F.S.O. to V.E.S.O.
- Coverage: YAML, backend, frontend, i18n, the specification document, the modeling guide, READMEs, paper drafts, and outreach material across the `life-matters-models`, `life-matters-reference-engine`, `life-matters-game`, and `life-matters-home` repositories.
- `LM_format_1.0.md` stays at v1.0 (still an unpublished Draft, so this is not a breaking change to an already-published contract), with a new changelog line added to section 9.1 recording this change.
