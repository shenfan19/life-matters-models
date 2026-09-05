# 0098 - optimization.schedules Adds mode: sustained (a Sub-Day-Step-Size Sustained Input)

**Date**: 2026-06-11
**Status**: Implemented
**Category**: Simulation engine / optimizer schema

---

## Background

`optimization.schedules`'s default scheduling is pulse mode (`regimen_runner.apply_regimens`): at the start of every step, every controlled variable is first reset to zero, and only the one step matching `time: "HH:MM"` is written with `value`.

For a model with `step_size: day` (or longer, an exact integer multiple of 86400 seconds), combined with `days: [Mon..Sun]` all checked, this mechanism is equivalent to "in effect every day," and can express a day-level sustained input.

But for a model with `step_size: hour`/`minute`, a single `time: "HH:MM"` schedule is in effect only during one specific hour or minute each day, and the remaining 23-plus hours have the variable zeroed out. This makes it impossible to express a decision variable such as sustained intensity, sustained protection level, or a sustained rest fraction, which should stay constant across several consecutive steps. This limitation was found while migrating a `step_size: hour` (360-step) scenario file's deprecated `variables_to_optimize`/`maps_to` schema.

## Decision

Add two optional, backward-compatible fields (omitting them preserves the original pulse semantics):

- `mode: sustained`: a scheduled event no longer depends on a single `time` trigger point; as long as the `days`/`date_range` filter is satisfied, the step is in effect (its value is not zeroed). The top-level `time:` field is ignored in this mode.
- `time_range: ["HH:MM", "HH:MM"]` (optional, used together with `mode: sustained`): further restricts the effective time window within each day, for a sub-day interval that does not span the whole day (such as the `[0,8)` hour window).

Files changed: `sim_engine/src/regimen_runner.py` (a new sustained branch added to `apply_regimens`), `sim_engine/src/optimizer_engine.py` (`_build_regimen_events`/`fixed_events_map` pass `mode`/`time_range` through).

Documentation: a new "mode: sustained" subsection added to `docs/model.md`.

## Validation

In an internal scenario file, after changing three decision variables (one split into two segments by `date_range`, plus two others) to `mode: sustained`, running `--opt` reached `feasible: 100%` from generation 3 onward, and the Pareto front showed a genuine two-objective trade-off.

## Scope / Follow-On

- Applies to schema migration for the remaining sub-day-step-size models (`ad079_it_pompeii`, `ad1666_uk_issac_newton`):
  - ad079's `[0,8)`/`[8,19)` hour windows do not align with day boundaries and need `time_range`.
  - References such as `@ bedtime` in ad1666 that depend on runtime state are still beyond this extension's scope and need a separate design.
- `mode: sustained` only adds a new branch and does not change the pulse behavior of existing models, so no existing YAML needs migration.
