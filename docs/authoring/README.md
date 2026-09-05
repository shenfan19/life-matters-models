# Modeling Practice Guide Index

`LM_format_1.0.md` is the formal, versioned format specification, defining the conditions a YAML file must satisfy to count as a valid LM file, and it is a baseline document that can be referenced independently. This directory sits on top of that specification and holds this project's practical guidance on how to write a good LM model, adjusted continuously alongside modeling practice rather than versioned, and it does not re-document field definitions already covered by `LM_format_1.0.md`; wherever a specific field structure is involved, it points directly to the corresponding section of the specification.

The directory is named `authoring` rather than `model` to keep it distinct from the repository's root `models/` directory, which holds the actual model YAML files, since two similarly spelled names would send readers to the wrong place; `authoring` corresponds to the act of writing a model, and the introduction to `LM_format_1.0.md` uses the word author to describe the same act, so the terminology is consistent.

## Contents

- [methodology.md](methodology.md): LM's core methodology, the judgment criteria for coupling rather than stacking, and the model inclusion criteria and discipline coverage inventory. Read this first when writing a new model or reviewing whether an existing model should keep a given mechanism.
- [description_writing.md](description_writing.md): The writing conventions for `metadata.description` and the top-level `references`, covering how the four fields `problem`/`method`/`result`/`limitations` divide responsibilities and how to write source-literature attribution. This file is self-contained and can be handed on its own to another collaborator or another AI session as material for writing a model description.
- [ratings.md](ratings.md): The `metadata.ratings` scoring system, a 0-1 scale, with scoring criteria for the technical fields (VESO: `variable`/`equation`/`simulation`/`optimization`) and the shared non-technical fields (`importance`/`innovation`/`confidence`).
- [bookkeeping.md](bookkeeping.md): The `metadata.todo`/`metadata.log` fields, the `history/` directory, and the `reviewed` field, covering the conventions for marking a model file's status and tracing its changes from draft to publication.
- [variables_and_equations.md](variables_and_equations.md): The three variable types and the `evidence_type` conversion rules, and the execution details of equations such as `step_unit`/`priority` and Euler discrete integration, going into more detail than the specification's own field definitions and including design rationale and known pitfalls.
- [regimens_and_optimization.md](regimens_and_optimization.md): The complete usage of `simulation.plans[*].regimens` and `optimization`, including positive and negative examples, the `lm_score` healthy-duration core metric, the T1-T4 decision-variable tiers, and the embedded format of `optimization.results`.
- [imports_and_organization.md](imports_and_organization.md): The `imports` merge rules and the model classification directory conventions.

## Division of labor with LM_format_1.0.md

Whether a field is legal, what values it can take, and whether it is required or optional are determined by `LM_format_1.0.md`; the files in this directory take these field definitions as already settled and supply guidance on how this project uses them well, which patterns are recommended, and which are known to cause trouble. Whenever the two conflict, `LM_format_1.0.md` governs, and the conflict should be treated as a sign that this directory's documentation needs correcting, not that the specification should accommodate an old convention.
