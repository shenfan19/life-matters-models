# 0137 - Merging the Top-Level evidence: Section into variables:, Switching to an evidence_type Field

**Date**: 2026-07-24
**Status**: Accepted

---

## Background

ADR 0040 designed a literature effect size (OR/HR/RR/Cohen's d, etc.) as an independent top-level YAML section, `evidence:`, sitting alongside `variables:`: an entry under `variables:` declares `type: state|input|parameter`, while an entry under `evidence:` declares `type: rr|or|hr|ard|cohens_d|ir|beta|pk`; at load time the loader converts an `evidence:` entry and writes it into `self.variables` (as `parameter` type), so the two YAML sections ultimately merge into the same runtime namespace, an `evidence:` entry is essentially a variable, just with one extra conversion step.

While reviewing LM format's top-level structure, the user raised: the project wants the YAML top-level categories to stay Unix-style, minimal and clearly at the same level (the `metadata`/`variables`/`formulas`/`sim`/`opt` line is clean, each doing one thing, and room for future expansion should be forced into an existing peer structure rather than continually stacking parallel concepts at the top level the way Windows does). `evidence:` becoming a sixth independent top-level section was exactly the start of this kind of stacking: it describes the same kind of thing as `variables:` (a named quantity in the model), only with the extra property that "its data source needs converting," yet it had been promoted into a fully independent top-level namespace, requiring an extra validation rule to keep the two namespaces from conflicting by name, "must not share a name with `variables:`," and that validation rule is itself evidence that there should not be two namespaces.

Two merge approaches were compared in discussion:

1. Encode the 8 evidence subtypes into `variables.type` itself (such as `type: evidence-ard`), expanding `type`'s enum from 3 values to 11.
2. Keep `variables.type` at 3 values (`state`/`input`/`parameter`) unchanged, and add an orthogonal field, `evidence_type`, to express the literature-effect-size subtype; an entry declaring `evidence_type` must have `type: parameter`.

Approach 1 would bind two orthogonal dimensions, the variable's role in the dynamics (the role dimension: state/input/parameter) and the statistical effect-size type of its data source (the source dimension: rr/or/hr/...), into a single string field, and directly overturn the decision, already argued for in ADR 0040's "Implementation Record," not to create a fourth type for evidence, to avoid preemptively expanding the type set for a not-yet-existing Modeller optimizer; approach 1 does not avoid creating a fourth type, it creates eight, expanding in exactly the opposite direction. Approach 2 keeps `type` at 3 values, and `evidence_type` is simply the first of the two provenance fields (`evidence_type`/`evidence_raw_value`, already defined in ADR 0040) that the loader would already auto-attach after conversion, moved forward from "a runtime artifact for lookup only" to "an input field the modeler fills in when declaring it," not a new concept in itself.

## Decision

Adopt approach 2: remove the top-level `evidence:` section, and move evidence declarations into `variables:` entirely, expressed as `type: parameter` plus `evidence_type: <one of 8 subtypes>`.

- A `variables:` entry gains an optional field, `evidence_type` (`rr`/`or`/`hr`/`ard`/`cohens_d`/`ir`/`beta`/`pk`). An entry declaring this field has `value` set to the raw literature value, and `type` must be `parameter` (otherwise the loader raises an error and rejects it); at load time the loader converts `value` in place into a coefficient usable in a formula (overwriting under the same name, no suffix added), with the pre-conversion raw value kept in the runtime `evidence_raw_value` field. The conversion formulas, the detailed treatment of the 8 subtypes, and the `applies_to` mechanism for automatically wiring into dynamics are all completely unchanged, only the data source switches from `data['evidence']` to `data['variables']` (see `docs/reference_engine/evidence/{conversion,applies_to}.md` in the `life-matters-reference-engine` repository for detail).
- The name-collision check between the `evidence:` and `variables:` namespaces (introduced in ADR 0040) is removed accordingly; once merged, that kind of conflict is already a duplicate key within the same YAML mapping, a YAML-syntax-level issue, no longer a semantic conflict this layer needs to check separately.
- A new check, not needed before and made necessary by this merge, is added: `applies_to` is only meaningful on an entry that declares `evidence_type` (before the merge, `applies_to` could only appear inside the `evidence:` section, so "written on a plain parameter" was not even possible; after the merge this misuse becomes possible, and the loader now explicitly rejects it).
- `VariableType` still has only 3 values, neither added to nor expanded, which is exactly a continuation, not a reversal, of ADR 0040's "Implementation Record" decision not to create a fourth type for evidence.

## Scope of Impact

**The `life-matters-reference-engine` repository**:
- `reference_engine/src/model_structure/loader.py`'s `_apply_model_data`: the two processing loops for `variables:` and `evidence:` are merged into one (with a new `evidence_type`/role-check branch added), the data source for the `applies_to` auto-wiring loop changes from `data['evidence']` to `data['variables']`, and a new check, "`applies_to` requires `evidence_type`," is added. `base.py` (the `Variable.evidence_type`/`evidence_raw_value` field definitions) and `routes/models.py` (the API response) are unchanged; both already worked under "the conversion result merges into `variables`" and are unaffected by this change.
- `docs/model.md`'s "Variable Types (3) + Top-Level evidence Conversion" chapter is rewritten in full as "... + In-Place evidence_type Conversion," including the YAML example, the `baseline_ref`/`applies_to` field table, and the complete schema example.
- `docs/evidence/conversion.md`, `docs/evidence/applies_to.md`: the conversion/wiring logic itself (the 8 subtypes' formulas, the validation order, the generated-expression templates) is unchanged, only the wording about "how to declare it in YAML" and the code references are updated (`data['evidence']` to `variables_data`), and a new "is `evidence_type` declared" precondition branch is added to the `applies_to` validation flowchart.
- `docs/design.md`, `docs/impl.md`: the evidence-related subsections' wording updated accordingly.
- `test_verification/errors/test_evidence_errors.py`: the scenario tested by the former `test_evidence_name_colliding_with_variable_is_rejected` (two namespaces sharing a name) structurally no longer exists after the merge, and is replaced by `test_evidence_type_on_non_parameter_role_is_rejected`, validating the new "`evidence_type` requires `type: parameter`" check.

**The `life-matters-models` repository**:
- `models/test_fixtures/valid/test_valid_evidence_{rr,or,hr,ard,cohens_d,ir,beta,pk,types}.yaml` (9 files) and `invalid/test_invalid_evidence_missing_baseline_ref.yaml`: the `evidence:` section moved into `variables:` in full.
- `invalid/test_invalid_evidence_name_collision.yaml`: the original test scenario structurally no longer exists, and is changed to test "declaring `evidence_type` on a non-`parameter`-role variable is rejected."
- `models/test_fixtures/fixture_catalog.md`: section 1's description text and this fixture's description updated accordingly.
- 13 real model YAML files under `models/references/**` (`class_size_2026`, `tobacco_elasticity_2026`, `mindfulness_mood_2026`, `mindfulness_anxiety_2026`, `inactivity_shortsleep_mortality_2026`, `cbt_depression_2026`, `processed_meat_mortality_2026`, `statin_ldl_doseresponse_2026`, `caffeine_pk_2026`, `waist_circumference_lungcancer_2026`, `smoking_lungcancer_2026`, `processed_meat_crc_2026`, `hepb_hcc_2026`): the `evidence:` section moved into `variables:`, each with one or two evidence entries, none involving `applies_to`/`baseline_ref`; after migration, each was reloaded and validated to confirm the converted value matches exactly what it was before migration.

**Historical ADR body text left unchanged**: ADR 0040 describes the design as it was when accepted (a top-level section) and its two "Implementation Record" entries; per the convention that historical records are not rewritten retroactively, its original text is kept, with no backfilling of this change; this ADR is its follow-on evolution record.

## Result

- The top-level YAML sections converge back from `metadata`/`imports`/`variables`/`formulas`/`simulation`/`optimization` plus `evidence` to the main line without `evidence`; an evidence declaration is now an optional dimension of a `variables:` entry, no longer an independent namespace.
- `VariableType` stays at 3 values, without introducing approach 1's 11-value role-times-source composite enum.
- After migrating the 9 test_fixtures plus 13 real model YAML files and reloading them, the converted values (including the 5 dynamics expressions `applies_to` generates automatically) were each checked one by one and matched exactly what they were before migration.

## Open Questions

- The GUI (sim_gui) has no dedicated visualization annotation for the `evidence_type` field yet (in the Overview table, a parameter converted from evidence currently displays the same way as a plain parameter, with no source badge), left to be filled in once the shared variables/formulas editor is implemented.
- If the paper (S1-S4) chapters or `LM_FORMAT_1.0.md` reference the old top-level `evidence:` section's syntax, they need a separate check and update (outside this ADR's engine/model change scope; tracked in a to-update note in `paper/c_paper_plan.md`).
