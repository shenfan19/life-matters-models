# 0126 - The Regimen-Semantics-Completeness Discussion: Three Scope Rulings (No Engine Change, Drawing the Line Between Model, Documentation, and Future Improvement)

**Date**: 2026-07-09
**Status**: Accepted; item 3 (sustained value equals the total across the whole effective window) has been superseded by [0131](0131-2026-07-13_model_sustained-value-per-day-not-per-span.md) (2026-07-13), changed to "each matching day independently delivers the full amount"; that item's text describes exactly the old rule 0131 now changes. Items 1/2/4 are unaffected.
**Category**: Modeling specification / regimen semantics

---

## Background

The internal task record `task_regimen-semantics.md` recorded four pending regimen-related questions (whether pulse-decay should become an engine feature, override versus accumulation for several regimen entries on the same variable, the decision rule for choosing between sustained and pulse-plus-decaying-state, and `valid_range` date alignment). The trigger case was a three-phase stacking bug found during an internal model-library review (see the internal task record `2026-07-07_issue_ibs_regimen_phase_stacking.md`).

This ADR records the conclusions from a discussion on 2026-07-09: one of these questions turns out not to belong to regimen design at all, one is explicitly ruled out as an engine or schema change, and one, after actually checking the code, turns out to be a pure documentation wrap-up.

## Decision

### 1. "A third pulse-decay semantics" is not a regimen/engine problem, it is a model-completeness problem

Checking a known-correct case in the internal model library confirmed: decay is entirely implemented by the `type: state` variable's own `dynamics` formula (of the form `X + eta_abs*input - k*X*step`), with the regimen only responsible for writing a constant via a pulse for the `input` variable, unrelated to decay. The engine has long supported any state variable writing its own first-order ODE.

Conclusion: a model that needs a decay effect just needs to copy this pattern, adding `type: state` plus `dynamics`; no schema or engine change is involved, and this is no longer treated as a pending "regimen semantics" question. The original direction, adding an engine `mode: pulse_decay` plus `half_life`, is not adopted.

Known precedent: the internal model-library review confirmed several models already use the correct pattern (a pulse input plus an independent decaying state), not individually listed here.

Newly found gaps of the same kind after reviewing the internal model library (not the design question of "should a decay formula be added" itself, but a bug that needs fixing in a specific model, recorded in the internal task `2026-07-09_issue_pulse-fed-instantaneous-formula-audit.md`):

- A blood-pressure-lowering effect term read the instantaneous pulse input directly instead of the accumulated state, sharing the same root cause as a previously fixed bug of the same kind, missed at the time. Fixed: changed to read the corresponding accumulated state.
- Another "progress-type" metric term read the instantaneous input directly, with a `step_size` of 1 hour running over several years, so the vast majority of hours read 0, possibly systematically inflating that metric. Recorded, not fixed (affects a paper's numbers, and needs a separate decision on the fix).
- A third case is structurally "in effect during the one hour it fires" rather than afterward, at a very small magnitude. Recorded, not fixed, low priority.

### 2. Override versus accumulation for several regimen entries on the same variable: no engine change, only the specific triggering file fixed

Three levels of investment were discussed: (a) load-time validation only (requiring that multiple entries for the same variable either all omit `date_range` or all declare it with no overlap), (b) a new explicit `phases` schema structure (with the engine guaranteeing partitioning), (c) a full redesign of how the input timeline is expressed (dropping implicit defaults such as "date_range omitted means the whole run").

Conclusion: none of the three is done. The project is just getting started and prioritizes stabilizing the sim baseline over a large engine or schema investment for the opt side's convenience or foolproofing; this kind of work is left for a future improvement or plugin. The status quo is kept: the engine's "reset to zero each step, then accumulate entry by entry" behavior is unchanged (this is correct as it stands; the three-meals accumulation is the intended design, not a bug).

The specific bug in `ibs_diet_opt_joint.yaml` (the T1 restriction-phase entry missing `date_range`, stacking across the whole run into T3/T4) is fixed only in that one file, and this is not treated as a model-specific special case; the general modeling convention below is used, applicable to any model:

Baseline-plus-increment convention (written into `docs/model.md`, not a mandatory rule): prefer designing "one baseline entry with no `date_range` (which should be effective for the whole run anyway) plus several increment entries scoped by `date_range` (whose value is the difference relative to baseline)," rather than "a complete target value written for each phase, with mutual exclusivity achieved end-to-end via `date_range`." In the former, the one entry with no `date_range` is meant to be effective the whole run anyway, with no "forgot to set an end date" trap; in the latter, every added phase requires that phase's `date_range` to not overlap the rest exactly, which is easy to get wrong.

A known limitation, accepted as a controlled cost: if a phase's switch timing is itself an optimizer search variable (such as T4), an increment entry's `date_range` endpoint needs to track T4's search value, and the engine currently does not support "one regimen's `date_range` endpoint equals another decision variable's decoded value." This kind of date-linkage need on the opt side falls back to manually fixing the search window, letting the modeler confirm the result on the Sim side and manually adjust it before re-searching if needed. There is no pursuit of "one optimization run automatically linking every phase boundary."

### 3. The decision rule for sustained versus pulse-plus-decaying-state: a pure documentation wrap-up, no longer waiting on question 1

Question 1 has been confirmed to involve no engine change, so there is no coupling of "if question 1 chooses direction A, this rule needs to change accordingly," and the decision rule already worked out in the S3 session (2026-07-07) can be written directly into the documentation, with no further design discussion needed.

The rule itself (confirmed against `schedule_runner.py`'s actual logic): sustained's `value` represents the total quantity across the whole effective window; the engine precomputes, at load time (not at run time), `N_steps = the effective window's total duration / step_size`, and at run time each hit step is written as `value / N_steps`. This requires the effective window's total duration (the days `date_range` covers, times the `days` filter, times the daily time segment) to already be a determinate number at load time.

The decision rule: is this input's effective-window length already a determinate value at the time the YAML is written or loaded?
- Yes (an accurate day count can be computed without depending on the optimizer's search result, such as a fixed "Monday through Friday"): use sustained.
- No (the window length itself is a T3/T4 search variable, or depends on runtime state): sustained cannot work (`N_steps` cannot be computed at load time); switch to "a pulse trigger, feeding into a `type: state` decaying variable, read by a downstream formula" (a purely local, step-by-step recurrence mechanism that does not need to know in advance how many steps there will be).

### 4. `valid_range` date alignment: keep the original judgment, handled independently

The mismatch between `_EPOCH = 1900-01-01` and a scenario's own `start_date` is low priority and narrow in impact (non-paper historical/game scenario models), and continues to be handled independently, out of this discussion's scope. This ADR does not implement this item.

## Result

- Fixed: the blood-pressure-lowering-effect formula in a case in the internal model library (see that file's `metadata.todo`/`log` entry dated 2026-07-09). `--sim`/`--opt` still needs to be rerun (both the sim file and its opt files import this file), and the corresponding paper numbers need updating.
- To be updated: `docs/model.md` needs the baseline-plus-increment convention, the opt-side date-linkage limitation (with an example), and the sustained decision rule added in three places (the next step after this ADR lands, not expanded within this ADR file).
- Recorded: the internal task `2026-07-09_issue_pulse-fed-instantaneous-formula-audit.md` (two pending bugs).
- The triggering case's specific fix is not implemented; it awaits the modeler applying the baseline-plus-increment convention above, after which the `--opt` numbers need re-validating and the corresponding paper updating.

## Open Questions

- The fix for one "progress-type" metric has not been chosen and needs a separate decision (see the linked task file above).
- The actual fix for the triggering case's file has not been implemented.
- Item 4 (`valid_range` date alignment) is not implemented and remains an independent, low-priority pending item.
- Paper-writing principle: this round's decision process and bug fix are not written into the paper, only recorded in the ADR and the corresponding model's `metadata.todo`; the paper presents only the corrected, revalidated final numbers.
