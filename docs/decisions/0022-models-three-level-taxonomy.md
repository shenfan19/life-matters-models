# 0022 - The Models Three-Level Classification System

**Status**: Implemented
**Date**: 2026-04-12
**Author**: shenfan19

---

## Background

As the model library (`models/references/`) grew, the original flat or shallow directory structure became hard to maintain. Main problems:
- Second-level directories such as `nutrition/`, `fitness/` lacked further classification.
- The `panic/` directory under `social/` was semantically unclear.
- A newly created model had no clear "home," leaving a contributor unsure where to place it.

## Decision

Adopt a three-level classification system, following these rules:

| Level | Meaning | Example |
|------|------|------|
| L1 | Major domain | `medical/`, `social/` |
| L2 | Discipline branch | `nutrition/`, `fitness/`, `economy/` |
| L3 | Subdivision | `nutrition/food/`, `nutrition/diet/`, `fitness/individual/` |

Not mandatory to go all three levels deep: an already-mature library-component directory such as `physiology/` stays flat, to be subdivided later once its file count grows, rather than adding a level for its own sake.

## Structure Snapshot (at Implementation Time)

```
models/references/
├── medical/
│   ├── physiology/           (flat, library components, not yet subdivided)
│   ├── nutrition/
│   │   ├── food/             (single-ingredient physiological models)
│   │   └── diet/             (dietary patterns and intervention plans)
│   ├── fitness/
│   │   ├── individual/       (individual endurance sports: running, swimming)
│   │   ├── team/             (team sports: basketball, soccer)
│   │   └── racket/           (racket sports: tennis, table tennis)
│   ├── disease/
│   │   ├── metabolic/        (metabolic disease: diabetes, obesity)
│   │   ├── chronic/          (chronic disease: liver disease, hypertension, CKD)
│   │   ├── acute/            (acute disease: influenza)
│   │   ├── infectious/       (infectious disease, to be filled in)
│   │   ├── mental/           (mental health, to be filled in)
│   │   └── genetic/          (genetic disease, to be filled in)
│   ├── medicine/
│   │   ├── pharmacology/     (pharmacokinetics, to be filled in)
│   │   ├── therapy/          (treatment plans, to be filled in)
│   │   └── preventive/       (preventive medicine, to be filled in)
│   └── surgery/
│       ├── orthopedic/
│       ├── cardiovascular/
│       └── general/
└── social/
    ├── economy/
    │   ├── labor/            (labor models)
    │   ├── market/           (to be filled in)
    │   └── finance/          (to be filled in)
    ├── conflict/
    │   ├── war/              (war dynamics)
    │   ├── disaster/         (natural disaster)
    │   └── civil/            (civil unrest, to be filled in)
    ├── law/
    │   ├── policy/
    │   ├── criminal/
    │   └── civil/
    ├── psychology/           (formerly panic/, with expanded scope)
    │   ├── panic/
    │   ├── behavior/
    │   └── cognition/
    ├── technology/
    │   ├── innovation/
    │   ├── infrastructure/
    │   └── digital/
    └── demography/
        ├── population/
        ├── mortality/
        └── migration/
```

## File Path Convention

Any cross-directory reference inside `imports:` must use the complete path starting from the `models/` root:

```yaml
imports:
  - components/medical/physiology/glucose_regulation   # Correct
  - glucose_regulation                                 # Only works within the same directory
  - medical/physiology/glucose_regulation              # Wrong: missing the components/ prefix, fails to resolve
```

## Follow-On Impact

- `physiology/` files stay flat and are not moved (to avoid a large-scale import-path change).
- The `standalone: false` marking on library components is unchanged.
- A newly added domain (such as `social/demography/`) reserves its directory, to be filled in as modeling proceeds.

## Rejected Alternatives

- Fully flat: every model placed one level under `medical/`. Poor scalability, hard to tell dozens of files apart.
- A four-level classification: too deep (such as `medical/physiology/endocrine/insulin/`), raising the maintenance cost of import paths.
- Classification by pathological process rather than discipline: such as `glycolysis/`, `immune_response/`, unintuitive for a cross-disciplinary researcher.
