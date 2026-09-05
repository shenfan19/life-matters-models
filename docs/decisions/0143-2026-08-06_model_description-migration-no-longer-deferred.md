# ADR 0143 - Migrating the description List Structure and references Contribution Notes Is No Longer a One-Time Scope Decision

## Status

Implemented

## Date

2026-08-06

## Background

ADR 0142 set the four-field list form for `papers/` model description and the two-field `{citation, description}` form for `metadata.references` as the writing convention, but at the time it was only actually applied by rewriting two files under `s1/infant_breastfeeding`, and its "Non-Goals" section explicitly stated that "the rest of the models under `models/papers/` will not be rewritten retroactively in bulk... migrating historical models is separate follow-up work."

That statement was describing the scope of that particular task: ADR 0142's own decision process was still trying out three different forms, and it had only validated the new form on the two `infant_breastfeeding` files, not yet ready for a broader rollout. But once that sentence was left in the ADR's body text, later tasks treated it as a citable, standing exception. A task that only did mechanical citation checking (on four models: S2's masld_insulin/ibs_diet, S3's smoking_stress, S4's hypertension_gout) cited this "Non-Goals" statement as a reason not to migrate those four models' `description` to the new format or their `references` to the two-field form, even though that same task was already adding citations to those models. This runs counter to the original intent that what each source publication specifically contributes should be written out as a matter of course; adding a citation without writing out its contribution amounts to doing the same job twice.

## Decision

1. Retract the "no retroactive bulk rewrite" statement in ADR 0142's "Non-Goals" section; it no longer serves as a reason to skip migration going forward.
2. Set the trigger explicitly: from now on, editing a `papers/` model file for any reason, even just checking a citation, fixing one field, or fixing a bug, should also migrate that model's `description` to the four-field list structure ADR 0142 defines and migrate `metadata.references` to the `{citation, description}` two-field form, with no need to wait for, or require, a dedicated bulk-migration task as a precondition.
3. Upgrade the `references` `{citation, description}` form from an "optional annotation" to the recommended default form for `papers/` models: what specific mechanism or data each source publication contributes is exactly this model's value relative to a single source publication and the most direct basis a reviewer has for checking whether the coupling actually holds, so it should be written out as a matter of course rather than treated as an optional nicety.
4. Historical models not yet migrated are not rewritten in bulk immediately because of this ADR; migration continues to happen naturally in the order files are touched. But from this ADR onward, "this task's scope does not include migration" is no longer a default reason to skip it; skipping is allowed only when the user explicitly states in a specific task that its scope excludes migration.

## Impact

- `docs/authoring/description_writing.md` gains a note on the migration trigger, and its wording for the `references` object form changes from "optional" to "recommended default."
- Any subsequent task touching a `models/papers/` model should by default include a description/references migration, unless the user states otherwise.
- The remaining models under `models/papers/` other than `s1/infant_breastfeeding` (S1's banister/ckd_protein/diuretic_tradeoff/fatty_liver/homair_ogtt, and all of S2/S3/S4's models) are migrated under this rule as they are touched over time.

## Non-Goals

- This does not change the substance of ADR 0142's four-field structure, its list-writing rule, or its citation format (a sentence-final parenthetical, no numbered references); those rules stay the same, and only their "no need to apply retroactively" exception is removed.
- This does not require rewriting every historical model in one concurrent batch; migration proceeds model by model as each is touched by a task.
