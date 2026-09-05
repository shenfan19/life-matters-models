# 0127 - Unifying input Variables to Sustained (No More Separate pulse Mode), Window Width Defaulted by Explicit Rule

**Date**: 2026-07-09
**Status**: Implemented
**Category**: Simulation engine / regimen semantics

---

## Background

ADR 0126 recorded three scope rulings from the regimen-semantics-completeness discussion, including question 2 (override versus accumulation for several regimen entries on the same variable), where the decision was to make no engine change and only fix specific files using the baseline-plus-increment convention. During that discussion, two complementary alternative directions were identified (recorded as direction 2 in the internal design backlog): one is adding an explicit allowlist to `type: input` (not adopted this round, see ADR 0126); the other is retiring "pulse" as a separately named concept and folding it into sustained (`N_steps=1` is already a special case of the same ADR 0099 rule, an equivalence already established by ADR 0099/0100), with the window width either explicitly declared by the modeler or defaulted by a rule.

This ADR records the second direction's rollout decision: there is no longer a separate pulse mode; every `type: input` variable is handled uniformly as sustained, differing only in window width.

The discussion also questioned whether "unifying window width" would sacrifice an already-validated pharmacokinetic (PK) curve shape; testing `nicotine_plasma` (20 cigarettes/day, `eta_abs=1.8`, `k_nic=2.77/day`, t1/2=6h) at three window widths found:

| Window width | Peak | Trough | Daily mean |
|---|---|---|---|
| A single-point pulse (the old default) | 38.0 | 2.3 | 13.0 |
| Spread over the 16 waking hours | 17.7 | 6.6 | 13.0 |
| Spread flat over the whole 24h day | 13.0 | 13.0 | 13.0 |

Conclusion: the daily mean is identical across all three (ADR 0099's property that the total does not change with how it is split holds strictly), and the peak-trough oscillation still exists at a reasonable window width (it does not get "flattened into a constant"); how wide to make the window is the modeler's own scientific judgment (real smoking behavior is neither a single instantaneous spike nor a uniform rate spread across the whole day), and any distortion from an unreasonable choice of step size or window width is the modeler's scientific-design responsibility, not something the engine needs to guard against. This is a natural safeguard that needs no additional mechanism to enforce. An already-validated PK model (`nicotine_plasma`/`thiazide_level`/`allopurinol_level`) can simply keep using a narrow window (1 step), with zero numeric change.

## Decision

### 1. Abolish the implicit behavior of "a pulse defaulting to some arbitrary conventional moment"

The original engine behavior (three independent, mutually inconsistent implementations in `schedule_runner.py::_normalize_time_interval`, `loader.py`, and `optimizer_engine.py`): when `time_start`/`time_end` were omitted, they defaulted to `'08:00'` (in `schedule_runner.py`/`optimizer_engine.py`) or `'00:00'` (in `loader.py`), a purely engine-implementation-detail "conventional moment" the modeler never declared. This is exactly the problem the ADR 0126 discussion pointed out: a day-rate input (such as `caloric_deficit`) was forced into the framework of "triggers at some moment," creating an ambiguity about "whether this phase is in effect" that should never have existed.

### 2. A new window-width default rule (`resolve_time_interval`, in `schedule_runner.py`)

| Form | Effective window | Applicable scenario |
|---|---|---|
| Neither `time_start` nor `time_end` written | All day, `["00:00","24:00")` | A day-rate input, with no natural "trigger moment" |
| Only `time_start` written | `time_end = time_start` (a single-step window) | A discrete event (a meal, a dose), numerically equivalent to the old pulse |
| Both written | An explicit interval | A "sustained intensity"-type input in a sub-day-step-size model |

All three forms share the same engine mechanism (ADR 0099's `value/N_steps` accumulation rule), not three branches; this is exactly what "unified" literally means: there is no "pulse branch" and "sustained branch" as two separate code paths, only a single function, `resolve_time_interval`, deciding the window width.

### 3. Code changes

Added `schedule_runner.py::resolve_time_interval(entry) -> (time_start, time_end)`, implementing the three-tier default rule above, replacing the previously scattered, mutually inconsistent `.get('time_start', '08:00'/'00:00')` forms across three files:

- `schedule_runner.py`: the internal calls in `precompute_sustained_divisors`/`apply_schedules` switch to the new function (the former `_normalize_time_interval` renamed and rewritten as a public function).
- `model_structure/loader.py::_parse_schedule_entries` (the `simulation.plans[*].regimens` parsing path): switched to call `resolve_time_interval`.
- `optimizer_engine.py` (the `optimization.startpoint.regimens` parsing path, in the `fixed_events_map` construction plus `_build_regimen_events`'s `d0` decoding, 3 places total): switched to call `resolve_time_interval`.

Four places of default logic merged into one, eliminating the previously hidden bug surface where `loader.py` ('00:00') and `optimizer_engine.py` ('08:00') were two mutually inconsistent paths.

### 4. Terminology: the documentation no longer uses "pulse" as a separately named mode

`docs/model.md` has been rewritten throughout: "pulse" and "sustained" are no longer listed side by side as two input types; the unified description is "sustained, with a window width that can narrow down to 1 step." Historical documentation (already-marked-deprecated passages such as `mode: sustained`'s old ADR 0098 format, the old `optimize.time` field, etc.) keeps its original wording, for reference when migrating an old YAML, and is not rewritten retroactively.

## Result

- Implemented: `reference_engine/src/schedule_runner.py` (added `resolve_time_interval`, replacing `_normalize_time_interval`), `reference_engine/src/model_structure/loader.py`, `reference_engine/src/optimizer_engine.py` (all three default-logic sites switched to the new function).
- Validated: all 23 pytest cases in `tests/` pass (including `test_schedule_runner.py`/`test_sim_cli_consistency.py`/every error-detection fixture in `tests/errors/`); a CLI smoke test (`hypertension_gout_sim.yaml --sim`, 5 plans x 5 MC runs) ran through with no regression.
- Updated: several places in `docs/model.md` rewritten (the regimens field description gains a "window-width default rules" table, the "unified interval representation" chapter rewritten, the "window width is a quantity of the same category" chapter rewritten), no longer introducing pulse as a separate mode.
- Repository-wide validation (added 2026-07-09, correcting a previously wrong statement below): it was believed that "writing no time at all" had never been legally used before; this turned out to be wrong on verification, an actual scan of all 205 `models/**/*.yaml` files found 18 real instances in `optimization.startpoint.regimens` (`ckd_protein_opt_*` x 4 `dietary_protein` files, `infant_breastfeeding_opt_*` x 3 files with 2 occurrences of `breast_milk` each, `bergman_glucose_opt_*` x 3 `exercise_met_min`, `masld_insulin_opt_*` x 3 `exercise_met_min`, `test_opt_t2.yaml` 2 occurrences), all sharing the same structure: T2 active (a 1-dimensional `optimize.time_start` search), with the base entry writing neither `time_start` nor `time_end` at all. Checking each one individually (actually running `--opt` once, pop=4/gen=1) confirmed behavior is unchanged: such an entry's final `time_end` is computed by `_shift_time` on the searched `time_start` plus `_width_min`; under the old default `_width_min=0` (a pulse, zero width), and under the new default `_width_min=1440` (all day), but `_shift_time` takes the modulus of 1440 minutes, `% 1440`, so both give the same result, the searched moment, zero width. This is a coincidence of the modulus arithmetic over 24 hours, not a design guarantee, and is recorded here for reference when investigating a similar issue in the future. Beyond this, running `--sim`/`--opt` once over all 205 files in the repository (papers 58 + test 45 + scenarios 27 + references 75; `--opt` run with a very small pop/gen to compress validation time) found zero failures caused by ADR 0127; the 17 failures found were all pre-existing, unrelated problems (a broken import plus content bugs in 14 files under `references/medical`, an old optimization schema in 2 files under `scenarios/` never migrated, and one isolated `test/valid` fixture missing an `optimize:` block), recorded in the internal task `2026-07-09_task_reference-library-broken-imports-audit.md`, outside this ADR's scope to handle.

## Open Questions

- Direction 1 already recorded in ADR 0126 (a `consumed_by` allowlist, solving the class of bug where the wrong physical quantity is read, such as `bp_dynagmics` needing to read `body_weight` but reading `caloric_deficit`) is complementary to this ADR, not part of it, and remains in the internal design backlog.
- The historical/deprecated passages in `docs/model.md` (the old ADR 0098 `mode: sustained` format description, the old `optimize.time` field mapping table) keep their original "pulse" wording and are not rewritten retroactively; if these passages ever need a full cleanup, that is a separate documentation-maintenance task, outside this ADR's scope.
- Whether "sustained-quantity" inputs (an accumulating metric such as `lm_score`, whether it too should be brought into this window-width unification discussion) was identified as a new question in the 2026-07-09 discussion, explicitly recorded as not solved in this first version; see the internal task `2026-07-09_task_cumulative-quantity-input-design.md`.
- The 17 pre-existing failures incidentally found during the repository-wide validation (a broken import under `references/medical`, etc., unrelated to this ADR) are unfixed; see the internal task `2026-07-09_task_reference-library-broken-imports-audit.md`.
