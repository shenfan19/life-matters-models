# 0086 - Format Naming: Life Matters Format (LMF) to Life Matters Model Language (LMML)

**Status**: Implemented
**Date**: 2026-05-26

## Background

The project's format was previously named **Life Matters Format (LMF)**. Comparing this against naming conventions in the same field (SBML, CellML, NeuroML) surfaced two problems:

1. "Format" is too generic: it conveys no information about the format's type (SBML uses "Markup Language" to state its structure clearly).
2. The scope is inaccurate: "Simulation Language" would cover only simulation, but this format also contains an `optimization:` block, so it is a complete description of a **model**, not a description of a particular execution mode.

The "Life Matters" brand name stays unchanged; its double meaning as both noun and verb ("matters" as important things, and "life matters" as life being important) is a deliberate design choice and gives it distinctive recognition in international research naming.

## Decision

The format's full name changes to **Life Matters Model Language**, abbreviated **LMML**.

The software/tool brand stays **Life Matters** (abbreviated LM), with no suffix.

| Level | Name | Abbreviation |
|------|------|------|
| Project/software brand | Life Matters | LM |
| Model format | Life Matters Model Language | LMML |
| Execution mode | Simulation / Optimization | — |

## Reasoning

- "Model Language" matches the naming pattern of CellML and NeuroML, carrying more professional recognition.
- "Model" accurately describes the format's scope: variables, equations, simulation configuration, and optimization configuration all belong to the model-definition layer and are not tied to a particular execution mode.
- The abbreviation LMML carries more semantic information than LMF.
- Naming the software and the format separately (LM versus LMML) matches field convention (such as Chrome versus HTML).

## Consequences

- LMF is uniformly replaced with LMML in paper titles and abstracts.
- LMF is replaced with LMML in model YAML description fields.
- LMF is replaced with LMML in internal planning documents.
- The `lm_score` and `lm_*` variable names are unaffected, since these are software/framework-layer naming, not a format abbreviation.
- Historical ADRs are left unchanged (ADRs are not edited retroactively).
