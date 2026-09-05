# ADR 0065 - metadata.description Supports Both a Structured Form and Free Text

## Status

Implemented (narrowed to three fields for `papers/` by ADR 0097; this ADR still applies to `references/` and other models)

## Date

2026-05-08

## Background

A model needs a reader-facing explanation, but a single long string easily mixes together the need, the problem, the method, the result, and the limitations. On the other hand, forcing every model to fill in fixed fields makes a simple model feel tedious and leaves many empty rows in the GUI.

What is needed is a flexible description rule: an author can write a single block of free text, or fill in structured fields as needed, and the interface displays only what is actually present.

## Decision

`metadata.description` supports two forms:

- A string: displayed as a `Brief`.
- A mapping object: every non-empty field is displayed in the order it appears in the YAML.

The recommended structured fields are:

`brief`, `need`, `problem`, `method`, `simulation`, `optimization`, `result`, `conclusion`, `limitations`

These fields are only a recommendation, not a schema constraint. An author can add other English keys, such as `scope`, `cohort`, `assumption`, `usage`, `evidence`, `mechanism`, `time_scale`, `sources`. The GUI automatically converts an unknown key into an English label, so `expected_cohort` displays as `Expected Cohort`.

Scientific basis, literature interpretation, and modeling assumptions are part of a model's description, but should not use a catch-all `science_note`. They should be split into more specific description sub-fields, such as `evidence`, `mechanism`, `time_scale`, `sources`; `metadata.science_note` or a top-level `science_note` is no longer used.

When a source can be attributed to a specific variable or equation, prefer writing it into that entry's `reference` field; `description.sources` keeps only background sources at the scenario level that cannot be split out.

`brief` replaces `summary` as the first recommended field, since it reads more like a model card's short blurb and does not imply it must be written as a paper abstract. `result` replaces `expected_result`; if there is no actual result yet, the content can simply state an expectation.

## Impact

- A simple model can continue to use `description: "..."`.
- A complex model can use a structured description without needing to fill in every field.
- The Overview page is more compact: a field's name and content sit on the same line, and a missing field is not displayed.
- The backend validator accepts `metadata.description` as either a string or a mapping object.
- `published` models uniformly adopt a structured Chinese-language description using the `brief` and `result` fields.

## Non-Goals

- This does not introduce a multilingual description schema.
- This does not turn the recommended fields into a hard schema requirement.
- This does not display the full description structure on the report page; the report page still centers on result output.
