# 0138 - Model Inclusion Criteria and Discipline Coverage Inventory

**Date**: 2026-07-30
**Status**: Accepted

---

## Background

The LM project previously had no clear "inclusion criteria"; deciding whether a discipline or a claim is worth the modeling effort always relied on the modeler's ad hoc judgment, with no reusable screening tool. Discussing case by case, when writing a paper or report, which parts succeeded and which fell short easily loses focus, lacking both a judgment basis and a systematic way to answer the more basic question of which disciplines LM format currently applies to and which remain unclear.

`models/test_validation/validation_report.md` already has a mature evaluation methodology, including a six-stage validation process, a four-dimension attribution classification, and a `validation_confidence` scale, but evaluation happens after a model is already built. Inclusion criteria address the problem before that point; the same four-dimension attribution classification is already mentioned in the "Validation Framework (Guide)" section as usable for pre-screening, but it had not been distilled into an independent, actionable judgment tool, nor did it have corresponding discipline-level inventory data.

## Decision

Adopt a three-stage framework of inclusion, evaluation, and output; this ADR corresponds to the inclusion stage:

### 1. A four-question screen, mapped to LM's four points of use

- **var**, corresponding to `variables`/`evidence`: does the discipline's claim carry a transferable, specific numeric value, rather than only a directional description?
- **for**, corresponding to `formulas`: does the mechanism have a recognized functional form that can be encoded directly?
- **sim**, corresponding to `simulation`: once encoded, can it produce a trajectory comparable against independent data points?
- **opt**, corresponding to `optimization`: does the decision space contain a real multi-objective trade-off worth running an optimization over?

The first two questions decide whether it is worth encoding into LM at all; the latter two decide how far it can be used once encoded. When var/for pass but sim/opt cannot be used, the model still stands, only its use is limited to a mechanism/engine cross-check display.

### 2. The four questions land in two places, with separated responsibilities

- A new Inclusion Test subsection is added to the Scope section of `docs/LM_format_1.0.md`, carrying the formal English statement of the four questions, as the format specification's own operational definition of what topics fall within LM format's scope. This part is versioned together with the format specification, with stable content that does not need frequent updates.
- A new discipline coverage inventory table is added to `docs/model.md`, carrying the four questions' concrete application within this project's model library, that is, the actual inventory result for each major and minor discipline against the var/for/sim/opt four questions, with rows taken from the existing directory structure under `models/references/`. This part is a living document that changes with each round of model evaluation and is not part of the versioned format specification.

The two are deliberately kept separate: the specification definition's stability and the inventory data's volatility are two different kinds of content, and mixing them would saddle the format-specification file with a maintenance frequency it should not carry.

### 3. Notation convention: four states, not three

Reuses the three-state notation already in `validation_report.md`, pass, fail, and not applicable, and adds a fourth state, not yet evaluated, with an explicit distinction between "not applicable" and "not yet evaluated": the former is marked `-`, meaning checked and confirmed there is no evaluable object; the latter is marked `?`, meaning not yet checked. This distinction itself was the key finding of this discussion: the `for` column in the inventory table showed `?` far more often than `-`, indicating that the real bottleneck at present is a lack of time to verify, not the mechanism itself being nonexistent, and these two situations call for entirely different follow-on actions, the former needs validation effort invested, the latter means this direction does not work. Without this distinction, the project's own unfinished work could be misread as the field itself lacking usable material, and vice versa.

### 4. Symbol style: plain text symbols, no colored emoji

The inventory table initially reused `validation_report.md`'s emoji symbols, a green checkmark, a red cross, and a gray dash for pass, fail, and not applicable, later changed to the plain-text √, ×, -, with `validation_report.md`'s emoji symbols throughout replaced to match. Emoji look vivid but unprofessional in a formal methodology document of this kind, and plain-text symbols better fit a scientific document's tone.

## Scope of Impact

- `docs/model.md`: a new "Model Inclusion Criteria and Discipline Coverage Inventory" section added, containing the four questions, the notation convention, and the major/minor discipline inventory table against var/for/sim/opt.
- `docs/LM_format_1.0.md`: a new "Inclusion Test" subsection added to the Scope section, carrying the formal English statement of the four questions. In the process, several places where this file had fallen behind the current implementation were also found and fixed: the standalone top-level `evidence:` block had already been superseded by ADR 0137, the `_nosim`/`_noopt`/`_noref` filename suffixes had already been superseded by ADR 0120, and field names such as `simulation.schedules`/`optimization.inputs` had already been superseded by ADR 0109/0088/0127; see that file's Version History entries for 2026-07-30 for detail. These fixes were incidental findings of stale maintenance from this work and are not part of this ADR's own decision, but are recorded here for reference.
- `models/test_validation/validation_report.md`: emoji symbols throughout replaced uniformly with √, ×, -; a new item added to the top task list requiring `docs/model.md`'s inventory table to be updated after each round of validation.
- Removed the unimplemented section 8 Game Conversion Block from `docs/LM_format_1.0.md`, that is, the top-level `game:` field and the "Story" term. An audit confirmed that the Reference Engine, the GUI, and no model YAML implements or uses this feature, and leaving it in the specification amounted to externally promising a feature that does not exist. Also removed the multi-individual simulation item from section 9.3 Planned Extensions, that is, the idea of multi-agent simulation within a household, which does not match LM's established individual-scale, batch-execution scope; independent MC sampling does not involve multi-agent interaction, and this should not have appeared as a planning direction before now.

## Result

- Several preliminary patterns worth further study surfaced while compiling the inventory table, left for later dedicated analysis before formal publication.
- Applying the inclusion criteria was not limited to "should a new discipline be modeled"; it also retroactively exposed a methodological problem in how the inventory table itself was compiled, for example that a small number of evaluated cases cannot represent an entire minor discipline, which in turn forced the inventory table itself to add an "N/M evaluated" count to avoid overgeneralizing from a partial sample.

## Open Questions

- The inventory table currently only goes down to the minor-discipline granularity, such as the `medical/disease` level. Whether it needs to go further, to a tertiary discipline level, for instance distinguishing `medical/disease/chronic` from `acute`, or whether it needs to be used to decide the next batch of modeling priorities in bulk, is a natural extension of this table's use and is not pre-designed by this ADR.
- The "Complete YAML Schema" reference example in `docs/model.md`, around lines 255 to 468, still uses old field names from before ADR 0109/0088, namely `simulation.schedules`/`optimization.inputs`, contradicting the authoritative sections later in the same file; this is an independent documentation bug found incidentally during the audit, outside this ADR's scope, left for later handling.
