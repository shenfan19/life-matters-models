# ADR 0151 - Promoting references: to a Top-Level Block; the M I V E S O R Top-Level Key Order

**Date**: 2026-08-23
**Status**: Adopted

---

## Background

`references:` used to be a sub-field of `metadata:`, sharing a container with purely administrative identification information such as `name`/`version`/`tags`/`authors`/`updated`. Most of those administrative fields can be added or removed at any time without affecting a model's scientific validity, but `references:` is different: it is where the project-level hard rule that every citation must trace back to a real publication is actually enforced, a direct pillar of a model's credibility and academic checkability, closer in nature to `imports:`, since both are a model file's strong binding declarations to the outside world, one binding to other LM files and the other to literature sources. Burying `references:` inside `metadata:` made it easy to treat as an optional decoration of the same importance as `tags`/`updated`, and easy to overlook when drafting or reviewing a model file.

## Decision

### 1. Promote `references:` to a top-level block

Moved from `metadata.references` to a top-level `references:`; the schema itself is unchanged, still a list of `{citation, description}` objects:

```yaml
references:
  - citation: "KDIGO 2020 Clinical Practice Guideline for Diabetes Management in CKD. Kidney Int."
    description: "GFR decline rate and protein restriction threshold."
  - citation: "Bauer J et al. (2013) Sarcopenia in CKD. NDT."
    description: "Muscle loss rate under CKD."
```

### 2. Unify the top-level key order as M I V E S O R

`metadata` -> `imports` -> `variables` -> `equations` -> `simulation` -> `optimization` -> `references`, read as M I V E S O R. The four core V.E.S.O. actions sit in the middle, with `metadata`/`imports` as the leading identity and composition declarations and `references` as the literature-support table closing out the end, echoing the reading habit of a paper's body text, discuss the model itself first and attach the reference list at the end, rather than making the reader face a pile of context-free citation entries before seeing any mechanism.

### 3. The relationship between `references:` entries and the `reference:` field on `variables`/`equations` is unchanged

This ADR only changes `references:`'s top-level position, not its schema, and does not require the existing per-entry free-text `reference:` fields on `variables.<name>` / `equations.<name>` to point at an entry within this block; the two continue to exist independently with no mandatory link between them. Changing the `reference:` field to point at a key inside this block, thereby turning "does this citation actually exist" into a script-checkable, machine-verifiable constraint, is a worthwhile future direction, but it involves a change to the engine's read logic and a backward-compatibility strategy, and is an independent design decision outside this ADR's scope.

## Engine Impact

None needed. A check of the entire `reference_engine/src` directory found that neither `references` nor `checksum` is read or validated by loader.py, validator.py, or any other module; `metadata.references` was previously just an ordinary dictionary parsed into memory along with the rest of `metadata`, never specially accessed by the engine via its path. After the move to the top level it likewise is not specially accessed; this is purely an organizational convention change for documentation and model files, and does not affect any existing `--sim`/`--opt` behavior.

## Trade-Offs

Given up: the simplicity of `references` sharing a container with the rest of the administrative fields; the number of top-level keys grows from five (with `references` implicit) to six.

Gained: `references` gains a structural standing equal to `imports`, directly reflecting its role in the model-credibility system as strongly bound and not to be casually discarded, no longer buried among administrative fields that can be added or removed at will; this also reserves a clear structural position for a future enhancement such as having the `reference:` field point at a key within this block for machine-checkable citation authenticity.

## Related Work

- Existing model files' `metadata.references` need migrating to the top-level `references:`, a purely mechanical move that does not change the citation content itself; see `skill_agent/lm-model-calibration.md` for the bulk-migration approach.
- `LM_format_1.0.md` has been updated to match: the section 1.1 top-level structure example now uses the M I V E S O R order; the section 1.2 Metadata Block removes the `references` sub-block; a new, standalone References Block section has been added; the section 8.1 Required Fields entry `metadata.references` is changed to `references`; the following section numbers shift accordingly.
- This rewrite also checked for and fixed a separate, unrelated long-standing gap in `LM_format_1.0.md`: ADR 0104/0105 (2026-06-16, adopted) had already moved `step_size` from `metadata.step_size` to `simulation.step_size` plus a per-equation `step_unit`, but that decision had never been synchronized into `LM_format_1.0.md` (its version-history table did not even have a corresponding change entry); the document's Terms table, Metadata Block example, Equation Expression Language, Simulation Block, Optimizer Independence Principle table, the optimizer YAML example's comments, and section 6.1 were all still on the old field paths from before ADR 0104. This has now been corrected as part of this rewrite, bringing the documentation in line with an already-adopted ADR, not a new design decision introduced by this ADR.
