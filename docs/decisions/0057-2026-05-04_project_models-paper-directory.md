# 0057 - A `models/published/` Paper-Specific Scenario Directory

**Date**: 2026-05-04 (created); 2026-05-05 (updated: `researches/` to `papers/`)
**Status**: Implemented

---

## Background

As paper writing progressed, model scenario files ended up mixing two different natures:

- Fast CI scenarios (`test/`): short duration (1 week), stripped-down parameters, used for development debugging and quickly validating the engine.
- Formal paper scenarios: full duration (52 weeks for CKD, 16 weeks for Banister), full MC settings, reproducing the paper's numbers.

The two kinds were mixed together under `models/scenarios/test/`, all named starting with `test_`, with no way to tell which corresponded to which paper.

---

## Decision

### Directory naming

Initially named `models/researches/`, later renamed to `models/published/`.

Reason for the rename: `papers/` is more direct than `researches/`; a subdirectory named after a paper can be renamed to the paper's title once published, and a user can tell at a glance that this is a debugged, paper-bound model.

### Directory structure

```
models/published/
  paper1/    # Paper 1 - a JOSS software-tool paper
    fatty_liver_a1_p1.yaml
    banister_b3_p1.yaml        <- the full 16-week V2 protocol
  paper2/    # Paper 2 - a JAMIA clinical validation
    ckd_protein_a4_p2.yaml     <- the full 52-week version (migrated from test/)
    hypertension_gout_a5_p2.yaml
  paper3/    # Paper 3 - a JBI optimization-methods paper
    ckd_protein_pareto_a4_p3.yaml
    hypertension_gout_3obj_a5_p3.yaml
    smoking_stress_a6_p3.yaml
```

Filename rule: `{topic}_{case_id}_{paper_id}.yaml`, with the topic first so files on the same topic (such as `ckd_protein`) naturally cluster together in a directory listing.

Once a paper is published, its subdirectory (`paper1/`) can be renamed to the paper's short title (such as `banister_fitness_2026/`), making the repository self-documenting for external readers.

The `test/` directory is kept: `test_banister.yaml` (a 1-week fast CI run) and `test_glucose_meal.yaml` continue to exist as engine regression tests, out of the paper's scope.

---

## Reasoning

- Paper scenarios and CI test scenarios serve different purposes and should be clearly separated.
- `papers/paper1/b3_banister.yaml`, corresponding to the paper's section 5.2, is more readable than `test/test_banister.yaml`.
- Each YAML file's `metadata.paper` and `metadata.case_id` fields map directly to the paper.

---

## Consequences

- `models/in_process/test/test_ckd_protein.yaml` migrated to `models/published/paper2/ckd_protein_a4_p2.yaml`.
- Path references in `docs/validation.md`, `models/source/README.md`, and `models/scenarios/README.md` updated accordingly.
- `models/published/` is discovered automatically via `scan_models`'s recursive traversal (see ADR 0058) and shows up in the GUI's model list.
