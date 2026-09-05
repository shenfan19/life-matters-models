# 0100 - Unifying pulse/sustained into the Time Interval [start,end); the GUI Removes the Three-State full day / time / sustained Choice

**Date**: 2026-06-11 (supplemented 2026-06-16)
**Status**: Complete (the schema/engine/GUI/T2/CLI paths are all implemented; backward compatibility with the old fields has been removed)
**Category**: Simulation engine / optimizer schema / GUI

---

## Background

[0098](0098-2026-06-11_sim_optimizer-schedule-sustained-mode.md) introduced `mode: sustained` plus `time_range` as a second mechanism alongside pulse (a single-point `time`). This created a triple redundancy:

1. At the schema layer: a schedule entry had to choose between a single-point `time` and a `mode:sustained` plus `time_range` interval, two sets of fields, two sets of validation logic.
2. At the GUI layer: `SimSetupTab` already had a `time` toggle (a pulse single point), plus the newly added `sustained` toggle, plus the implicit third state of "not writing `time_range` means in effect all day." The three overlapped in meaning, and a user had to first understand the classification of "do I want a single point, an interval, or all day" before knowing which toggle to flip.
3. At the K×4 theory layer: an iCal-style calendar event is essentially a `[start, end)` interval (two time points), and a pulse compressing it to a single point is one special case, but the current state turned "the special case" and "the general form" into two parallel sets of fields.

## Decision

### Schema: a single pair of fields, `time_start` / `time_end`

Sub-day time is now uniformly expressed as a pair of `"HH:MM"` fields forming the interval `[time_start, time_end)`, replacing `time` plus `mode` plus `time_range`:

| Value | Meaning |
|---|---|
| `time_end == time_start` | Pulse: a zero-width interval, `N_steps=1` (per the 0099 formula, `value` is written as-is into that step) |
| `time_end != time_start` (not spanning the whole 00:00-24:00 day) | Sustained: each step in the interval gets `value/N_steps` (0099) |
| `time_start="00:00", time_end="24:00"` | All day: `[0,24)` full coverage, a specific value sustained can take, not a separate state |

The three are the same pair of fields taking different positions on the timeline, no longer three independent switches or branches; pulse, sustained, and all-day now uniformly go through 0099's `value/N_steps` path (pulse is simply the special case `N_steps=1`).

The `days` (day-of-week filter) and `date_range` (calendar interval) fields are unchanged, orthogonal to `time_start`/`time_end`.

### Backward compatibility (deprecated, removed 2026-06-16)

The old fields (`time`, `mode: sustained`, `time_range`) are no longer supported at all; the engine no longer maps them. An old YAML file must be updated manually, per this mapping:

| Old form (no longer supported) | Equivalent new form |
|---|---|
| `time: "HH:MM"` | `time_start: "HH:MM"` (omitting `time_end` defaults to the same value, i.e. a pulse) |
| `mode: sustained` plus `time_range: [a, b]` | `time_start: a, time_end: b` |
| `mode: sustained` (with no `time_range`) | `time_start: "00:00", time_end: "24:00"` |

### GUI: a single "start time" control plus an optional "end time"

- Every input event always shows a "start time" input box (replacing the existing `time` toggle).
- "End time" defaults to equal the start time, visually collapsed/greyed out (displayed as a single point, i.e. pulse); dragging or filling it in to a different value automatically expands it into an interval (i.e. sustained), with no separate "sustained" checkbox needed.
- Setting the end time to span `00:00-24:00` (i.e. `time_start="00:00", time_end="24:00"`) represents "all day," with no separate "all day" checkbox needed; all day is simply one value the interval width can take.
- Removed: the `sustained` toggle, the "single point only" meaning of the `time` toggle, and the implicit state of "leaving time_range blank means all day." The GUI's state count drops from "a combination of 3 independent toggles" to "the relative relationship between 1 pair of time fields," reducing the comprehension burden by one dimension.

### Impact on K×4 / x-vector encoding

- T1 (value): unchanged, its semantics changed by 0099 to "the total within the interval."
- T2 (time): unified from "a single point, 1 dimension, versus sustained's `time_range`, 2 dimensions (not optimizable)" into:
  - Optimizing only `time_start`, with `time_end - time_start` (the interval width) fixed: 1 dimension (the most common case, such as "what time does today's work start," with the work duration fixed).
  - `time_start` and `time_end` optimized independently: 2 dimensions (such as "both the start and end of care intensity are to be searched").
  - Both ends optimized, each with its own bounds: the opt input side becomes 4 numbers (`start_lo,start_hi,end_lo,end_hi`), corresponding to what the user described as "4 values."
- `docs/model.md`'s x-vector encoding table (lines 927-938) needs a new branch for "whether the interval width itself participates in optimization"; most scenarios have a fixed width and add no dimensions, and only an entry explicitly declaring "optimize the duration" enters the 2/4-dimension branch.

## Implementation Precondition and Scope

Depends on 0099 being implemented first: the `N_steps` division logic is this ADR's foundation (pulse reuses the `N_steps=1` path), and 0099 has already required the "precompute N_steps" change points (regimen_runner plus its callers); this ADR only needs to switch "which fields the interval is read from" to `time_start`/`time_end` on top of that, with no need to redesign the computation path.

Scope involved (handled item by item during implementation):

- `sim_engine`: `regimen_runner.py` (interval parsing), `routes/simulation.py` (schema fields), `optimizer_engine.py` (x-vector encoding, `_build_regimen_events`).
- `sim_gui`: `types.ts` (the `InputEvent` fields), `Simulator.tsx` (YAML-to-state mapping), `SimSetupTab.tsx` / `OptSetupTab.tsx` (GUI controls), `optUtils.ts` (encoding), the various language locales.
- `docs/model.md`: the K×4 chapter, the x-vector encoding table, the "mode: sustained" subsection (removed/merged into the interval subsection).
- `papers/s5` (K×4 control theory): the terminology adjusted from "K×4" to "K×(a variable dimension)," or K×4 kept as the common-case description of "T1+T2 (start only)."

## What Is Not Done

- No fixed "minimum interval width" (such as 30 minutes) is introduced. The interval's minimum width is naturally defined by `step_size` (`time_end == time_start` gives `N_steps=1`), consistent with the "step only affects precision" principle, avoiding introducing a second time granularity unrelated to `step_size`.
- No "intensity/rate" concept is introduced; `value` is always "the total within the interval" (same as 0099), consistent with ADR 0092's bare-unit rule.

## Implementation Record (Schema/Engine Part)

- `regimen_runner.py`: added `_normalize_time_interval(ev) -> (time_start, time_end)`, mapping `time` / `mode: sustained` plus `time_range` / an explicit `time_start` plus `time_end` uniformly into a pair of `"HH:MM"` strings per this ADR's equivalence table; `_time_range_day_seconds` changed to accept `(time_start, time_end)`; both `precompute_sustained_divisors` and `apply_regimens` now branch on `time_start == time_end` (pulse) versus `!=` (sustained, including all-day), no longer relying on the `mode` field.
- `routes/simulation.py`: `RegimenEventData` gains `time_start`/`time_end` (`Optional[str]`), coexisting with `time`/`mode`/`time_range`, taking effect per parsing priority.
- `optimizer_engine.py`: `fixed_events_map` and `_build_regimen_events`'s `d0`/`ev2` pass `time_start`/`time_end` through (when the YAML entry provides them).
- `docs/model.md`: added the "unified interval representation: time_start / time_end" subsection (including the equivalence table and the backward-compatibility mapping); the `mode: sustained` subsection is marked as the old format but still supported; the x-vector encoding subsection notes that the T2 multi-dimensional redesign is not yet implemented.
- Validated: `models/test/test_sustained_mode.yaml` (`--sim`/`--opt`) and another internal scenario file (`--opt`) give numeric results matching before the change; no old YAML needed modification.

## Implementation Record (GUI Part)

- `types.ts`: `InputEvent` removed `time`/`timeEnabled`/`sustained`/`timeRangeStart`/`timeRangeEnd`, added `timeStart`/`timeEnd: string` (always has a value; equal means pulse, unequal means sustained, including `"00:00"~"24:00"` for all day).
- `simUtils.ts`: added `normalizeTimeInterval(raw)` (mirroring the backend's `_normalize_time_interval` equivalence table, converting YAML/session data to `{timeStart, timeEnd}`) and `migrateInputEvent`/`migrateInputEvents` (migrating an old localStorage session's `time`/`timeEnabled`/`sustained`/`timeRangeStart/End` to `timeStart`/`timeEnd`, with an already-migrated event returned as-is); `xToInputEvents`'s event matching, creation, and T2 slot write-back all switched to `timeStart`/`timeEnd`.
- `Simulator.tsx`: every YAML-to-state mapping point (`schedList`/`schedDict`/`plan.schedules`/`optimization.schedules` decision-item matching, session restoration, new-event defaults, Pareto labels) uniformly switched to `normalizeTimeInterval`/`migrateInputEvents`/`timeStart`/`timeEnd`.
- `optUtils.ts`: `buildOptSchedules` uses `isPulse = ev.timeStart === ev.timeEnd` to unify the three branches `mode='sustained'`/`time_range`/`time` into `entry.time_start`/`entry.time_end`; the T2 branch (`isPulse && ev.optimizeTime`) keeps `optBlock.time`/`time_step`, without sending `time_start`/`time_end` (to avoid conflicting with the backend's interval-parsing priority).
- `useSimulation.ts`: the regimen payload at all three sites unified as `{ id, time: ev.timeStart, value, time_start: ev.timeStart, time_end: ev.timeEnd }`.
- `SimSetupTab.tsx`/`OptSetupTab.tsx`: removed the "time"/"sustained" toggles and the "daily" hint, added an always-shown pair of "start time -> end time" controls; when `timeStart===timeEnd`, the end time is greyed/dashed (pulse), and editing the end time to a different value switches it to sustained, with a collapse button (x) to reset back to pulse. In `OptSetupTab.tsx`, T2 (the `opt` toggle plus time window plus step selector) is shown only in the pulse state, with its logic and field names unchanged.
- The 4 locales (en/zh-CN/zh-TW/fr): removed `tog.time`/`tog.time_tip`/`tog.sustained`/`tog.sustained_tip`/`setup.daily`, added `time_start_tip`/`time_end_tip`/`time_collapse_tip`.

## Implementation Record (T2 x-Vector Redesign)

- Schema: `optimize.time` renamed to `optimize.time_start` (the old name still supported as an alias). Writing only `time_start` gives 1 dimension (the interval width fixed, with `time_end` equal to the searched `time_start` plus the original width); additionally writing `optimize.time_end` gives 2 dimensions (start and end searched independently). A "4-dimensional (each with its own independent bounds)" form was not implemented; the "4 numbers" in the ADR draft correspond exactly to the two windows' respective `[lo,hi]` in the 2-dimensional scenario, not an additional dimension.
- `optimizer_engine.py`: added the helper functions `_hhmm_to_min`/`_shift_time`; `var_specs`'s `kind='time'` split into `'time_start'`/`'time_end'`; during decoding, `d0` precomputes `_width_min` (equal to the entry's own `time_end - time_start`) and `_time2dim` (whether `optimize.time_end` is declared); after decoding `time_start`, if not 2-dimensional, `time_end` is derived via `_shift_time`; the output `ev2` always carries `time_start`/`time_end` (plus the compatibility field `time = time_start`).
- `optUtils.ts`: `buildOptSchedules` always passes `entry.time_start`/`entry.time_end` through as the width template; `ev.optimizeTime` maps to `optBlock.time_start`; for a sustained event, a new `ev.optimizeTimeEnd` maps to `optBlock.time_end` (2 dimensions).
- `types.ts`: `InputEvent` gains `optimizeTimeEnd`/`timeEndWindowStart`/`timeEndWindowEnd`.
- `OptSetupTab.tsx`: the T2 `opt` toggle is shown for both pulse and sustained (no longer pulse-only); when sustained and `optimizeTime` is set, a new "end" (`time_end`) toggle row controls whether the interval's end is searched independently.
- `simUtils.ts`: added `hhmmToMin`/`shiftTime` (mirroring the backend); `xToInputEvents`'s T2 branch consumes 1 or 2 x components depending on 1 or 2 dimensions.
- `Simulator.tsx`: parses `optimize.time_start`/`time_end` from YAML to state (including compatibility with the old `optimize.time` alias).
- `docs/model.md`: the T2 subsection rewritten as a 1/2-dimensional schema description, with the x-vector encoding table gaining the corresponding branch.
- The 4 locales: added `sim.opt.tog.time_end`/`time_end_on_tip`/`time_end_off_tip`/`time_end_fixed_hint`.

## Implementation Record (Papers Part)

- After review, S1 section 3.2's formal definition $R_j = \{(t_k, d_k, p_k, v_k)\}$ and the $4KM$-dimensional search-space statement are themselves compatible with this ADR: $t_k$ (the start moment) is exactly `time_start`, and the sub-day effective width $w_k$ (`time_end - time_start`) is fixed in most scenarios, adding no decision variables. So the full rename from "K×4" to "K×(a variable dimension)" was not adopted; instead, a "supplementary note (the sub-day effective interval)" paragraph was added after S1 section 3.2's Definition 1, clarifying $w_k$'s meaning and noting that only when both the interval's start and end are searched independently does that segment contribute a 5th decision variable (a K×5 extension, a rare case). S3's Definition 2 gains a matching annotation pointing to this note. S2/S4 only reference K×4 informally and need no change. S5 (not yet drafted) will need to incorporate this extension when written.

Once the old fields are removed, an old YAML file needs manual updating (see the "backward compatibility" mapping table above).

## Implementation Record (the CLI Path, Supplemented 2026-06-16)

The original implementation only covered the API/GUI path (`apply_regimens`); the CLI path (running a simulation directly from a YAML file) used `_apply_schedules` plus the `InputSchedule`/`SchedulePoint` mechanism, which did not support `time_end` or sustained. This supplement brings the CLI path in line with the GUI, with both paths now going through `apply_regimens`:

- `model_structure/loader.py`: `_parse_schedule_entries` no longer expands absolute time points (`InputSchedule`/`SchedulePoint`), instead outputting a regimen-compatible list, with each entry becoming `{variable, events: [{time_start, time_end, value, days?, valid_start?, valid_end?}]}`. The result is stored into `model.plans[plan_id]` (a `List[dict]`), with the first plan also stored into `model.schedule_entries`. Reading of the old `time:` field is removed entirely (no longer backward-compatible).
- `model_structure/core.py`: added the `self.schedule_entries: list = []` attribute.
- `simulator_engine.py`: `run_simulation` calls `precompute_sustained_divisors` before the step loop, and each step calls `apply_regimens` before `model.step()`, fully symmetric with `session_manager.py`'s GUI loop; `run_simulation_all_plans` now sets `model.schedule_entries` instead of `model.schedules`.
- `_apply_schedules` is kept, but now handles only `daily_inputs` (absolute-time `InputSchedule` objects); plan-based schedule entries have moved out of `self.schedules` and are no longer double-counted.
- `docs/model.md`: the `mode: sustained` subsection marked "deprecated (no longer supported)"; the old-field mapping table's wording changed from "still supported" to "old form (no longer supported)."
- Validated: two test models, `test_plans` (three plans, pulse) and `test_sustained_mode` (pulse plus sustained, including a window spanning midnight, `20:00~08:00`), passed through `run_simulation_all_plans`, with results matching the GUI path's expectation (pulse over 5 days gives `cumulative_output=300`; sustained over 5 days gives 60).
