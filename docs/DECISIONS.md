# Model Decisions Summary

> This file is a topic-classified summary of the ADRs in `docs/decisions/` related to the YAML model format and the model library structure.
> ADRs for the simulation engine, optimizer, and UI are in `life-matters-reference-engine/docs/DECISIONS.md`.
> The complete chronological index is in [decisions/README.md](decisions/README.md).

Importance: ⭐⭐ = a core constraint, affecting the format specification or architecture, not to be changed lightly; ⭐ = an important implementation decision; unmarked = implemented, a historical record.

---

## 1. YAML Model Format (see model.md)

| ADR | Title | Importance | Status |
|-----|------|---------|------|
| [0040](decisions/0040-2026-04-22_sim_medical-evidence-types-and-variable-mapping.md) | **8 medical evidence subtypes (distinguishing evidence from parameter)** | ⭐⭐ | Done |
| [0044](decisions/0044-2026-04-30_sim_schedule-as-simulation-input-subtype.md) | **schedule belongs under the simulation block; pulse mode; discrete inputs need no zero-value point** | ⭐⭐ | Done |
| [0046](decisions/0046-2026-04-30_sim_step-size-design-metadata-and-formula-symbol.md) | ~~step_size metadata~~ (superseded by ADR 0104) | | Deprecated |
| [0104](decisions/0104-2026-06-16_model_step-unit-per-formula-and-sim-step-size.md) | **Per-equation `step_unit` (required) plus `simulation.step_size`; removes `metadata.step_size`** | ⭐⭐ | Done |
| [0053](decisions/0053-2026-05-03_sim_date_range-scheduling-field-and-yaml-schedule-priority-fix.md) | The date_range field; the YAML Schedule takes priority over the GUI Regimen | ⭐ | Done |
| [0063](decisions/0063-2026-05-07_sim_resolved-imports-and-output-selection.md) | **Resolved imports and the output-variable selection rule (later revised by 0107)** | ⭐⭐ | Done |
| [0107](decisions/0107-2026-06-16_model_output-variables-import-overwrite.md) | **`output_variables` / `output_types` import behavior unified to overwrite (replacing union)** | ⭐⭐ | Done |
| [0065](decisions/0065-2026-05-08_sim_structured-description.md) | metadata.description supports a structured form (brief/need/method and other fields) | ⭐ | Done |
| [0075](decisions/0075-2026-05-17_model_remove-type-standalone-fields.md) | **Removes the top-level YAML `type` and `standalone` fields** | ⭐⭐ | Done |
| [0092](decisions/0092-2026-06-05_model_input-variable-bare-unit-rule.md) | **The bare-unit rule for `type: input` variables (an event quantity, rate units prohibited)** | ⭐⭐ | Done |
| [0096](decisions/0096-2026-06-06_model_filename-quality-markers.md) | **Filename quality markers: the `_nosim` / `_noopt` / `_noref` suffix convention** | ⭐⭐ | Done |
| [0098](decisions/0098-2026-06-11_sim_optimizer-schedule-sustained-mode.md) | optimization.schedules adds `mode: sustained` (a sub-day-step-size sustained input) | ⭐ | Done (an old format, superseded by 0100 but still supported) |
| [0099](decisions/0099-2026-06-11_sim_sustained-value-step-invariance.md) | A correction to sustained mode's `value` semantics: window total divided by N_steps (step-size invariance) | ⭐⭐ | Done |
| [0100](decisions/0100-2026-06-11_sim_unify-pulse-sustained-time-interval.md) | **Unifies pulse/sustained into the time interval `time_start`/`time_end`; the GUI removes the three-state full day/time/sustained choice** | ⭐⭐ | Partially implemented (the papers/s5 terminology not yet updated) |

---

## 2. Model Library Structure and Naming

| ADR | Title | Importance | Status |
|-----|------|---------|------|
| [0022](decisions/0022-models-three-level-taxonomy.md) | **The three-level model classification system (medical/social to discipline to subdivision)** | ⭐⭐ | Done |
| [0041](decisions/0041-2026-04-22_project_naming-convention-underscore-preferred.md) | **Naming convention: snake_case with underscores preferred** | ⭐⭐ | Done |
| [0042](decisions/0042-2026-04-23_project_mod-to-model-rename.md) | A project-wide rename from mod to model | | Done |
| [0057](decisions/0057-2026-05-04_project_models-paper-directory.md) | The models/papers/ directory convention for paper-specific models | ⭐ | Done |
| [0062](decisions/0062-2026-05-06_project_models-directory-rename.md) | An update to the top-level models directory and file-naming convention | | Done |
| [0086](decisions/0086-2026-05-26_project_lmml-rename-from-lmf.md) | **Format naming: LMF renamed to LMML (Life Matters Model Language)** | ⭐ | Done |

---

## Maintenance Rules

- A new ADR: write it into `decisions/` and add a row to `decisions/README.md`, and also add a row to the corresponding category in this file.
- When a ⭐⭐ decision changes: update the corresponding section of `model.md` accordingly.
- ADRs about the YAML format go into section 1; ADRs about model-library organization go into section 2.
