# 0139 - An AI-Generated-Content Statement and the Author/Model-Library Responsibility Boundary

**Date**: 2026-07-30
**Status**: Accepted

---

## Background

The same tension kept resurfacing during the model library audit: the author is a single researcher, not a team, and not an encyclopedic expert across every field; even with AI assistance, expert-level verification of every model in every discipline in the model library is simply not achievable within one person's capacity. This had not previously been stated explicitly, leading to two downstream problems:

1. Trying to make the docs, `validation_report.md`, and the model library as a whole reach "comprehensively reliable" is, in fact, not achievable within one person's effort, and the discussion kept getting stuck in circles, making it hard to move the S1 paper toward publication.
2. During the public-repository audit, it became clear that even after extensive wording cleanup, the possibility that text could be taken out of context or invite challenge could not be fully ruled out, and an explicit statement was needed to draw a clear line between what the author is responsible for and what the model library as a whole is.

## Decision

Adopt two boundary statements as the project's basic framework for external communication:

### 1. The author's responsibility boundary, corresponding to the output stage of "inclusion to evaluation to output"

The author is personally responsible for, and only for the correctness of, the following:

- The LM format specification itself, that is, `docs/LM_format_1.0.md`.
- The accompanying simulation and optimization engine, that is, the life-matters-reference-engine repository.
- A small number of curated cases in the first paper, S1, two to three core demonstration models that have already reached a `validation_confidence` of at least 4.

`docs/model.md`, `models/test_validation/validation_report.md`, and the rest of the model library are allowed to contain imperfections or even errors. This is not a lowering of standards; these documents were designed from the start as living documents, meant to honestly record the current state of validation and invite experts across disciplines to review and revise them, and `models/test_validation/validation_report.md` states at the outset that its goal is not to cherry-pick a handful of working model examples but to establish a general diagnostic framework. This ADR makes this already-implicit positioning explicit as part of the project's external communication.

### 2. A statement on the model library's nature: AI-assisted generation, not expert-verified, for testing and reference only

Most model files in the model library are AI-assisted in generation and initial review, have not yet been verified by an expert in the relevant field, and do not constitute a clinical or scientific conclusion. Each model's `metadata.ratings.validation_confidence` field marks its current level of validation, with the scale defined in `docs/model.md`; `docs/model.md`'s inclusion-criteria inventory table and `models/test_validation/validation_report.md` honestly record which disciplines and which models have been validated and which are still pending evaluation. Experts in the relevant fields are welcome to point out corrections, which is exactly why the models are published in an open, checkable YAML format rather than locked inside a private tool.

Positioning analogy: the project is similar to a computable scientific reference library built for the AI era. AI's broad participation in the research workflow is already a reality, not a problem to avoid, and making AI-assisted content checkable and correctable is one of the reasons a project like this exists, not a shortcoming to hide.

### 3. A wording boundary: S1, a paper about a software tool, does not argue for the "AI feeding back into AI" long-term vision

The author has raised a larger vision, that LM supplies AI with computable, logically constrained feedback material to address the shortcomings of AI systems that currently rely mainly on DNN fitting with limited logical-reasoning capability. This vision has its own discussion value, but it is not S1's core argument. S1 is positioned as a focused, restrained software-tool paper, and if it carried a grand narrative mismatched with its actual contribution, the actual contribution being the YAML format and the CLI tool themselves, that would more easily invite a reviewer's challenge about ambition outstripping contribution, running counter to the very goal of reducing wording-related risk. This kind of vision is left for a follow-on paper positioned as a Perspective, namely S3 or S4, or a short outlook paragraph in S1.

## Scope of Impact

- `life-matters-models/README.md`: a new "Content Credibility Statement" section added.
- `life-matters-reference-engine/README.md`: the "Disclaimer" section gains a paragraph clarifying that the author is responsible for the format specification and the engine itself, with the model library's credibility boundary pointed to the life-matters-models README.
- `docs/model.md`: the inclusion-criteria section gains a sentence stating that the inventory table's many `?` cells honestly reflect the model library's current state of AI-assisted generation without expert verification.
- `models/test_validation/validation_report.md`: the opening introduction gains a matching sentence.
- The exact wording of the AI-use disclosure statement in drafts of S1, S2, and other papers needs to be separately aligned with the specific requirements of the target journal, such as the author guidelines of BMC Medical Informatics and Decision Making; this is not expanded on within this ADR and is tracked in the relevant task records under `life-matters-home/paper/`.

## Result

- The project's external communication line is unified: the author is responsible for the format, the engine, and S1's curated cases; the model library as a whole is an open, AI-assisted resource inviting multi-party review, and the two are not conflated.
- This provides a clear scope anchor for moving S1 and S2 toward publication: there is no need to wait for the model library to become "comprehensively reliable," only for S1's small set of curated cases to reach a strong validation standard, with the boundary stated clearly and honestly.

## Open Questions

- Whether an explicit AI-generation marker field, such as `ai_generated: true`, is needed in an individual model YAML's `metadata`, or whether the current approach of stating this once at the repository-level README/docs, without per-file annotation, should be kept. This ADR adopts the latter, simpler and cheaper to maintain; the former is kept as a future fallback if needed, not implemented now.
- The specific wording of the AI-use disclosure in the S1 paper's own text will be settled once the target journal's requirements are confirmed.
