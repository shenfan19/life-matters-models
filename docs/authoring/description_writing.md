# Writing Conventions for metadata.description and Top-Level references

### `metadata.description`

`description` supports two forms. A plain form:

```yaml
metadata:
  description: "A short explanation."
```

Or a structured form:

```yaml
metadata:
  description:
    problem: "The scientific question the model answers, and why a single source publication cannot answer it."
    method: "Which decision variable or resource-competition pathway the combined mechanisms share."
    result: "A summary of the simulation conclusion or Pareto front; if not yet run, state the theoretical expectation and note that it has not been run."
    limitations: "Known modeling boundaries and parameters that still need refinement."
```

The structured form is recommended to use English field keys. The set of fields is not fixed; the GUI displays every non-empty field in the order it appears in the YAML, and fields that are not written neither display nor take up blank space.

For the standard structure of `papers/` models, the `method` field was restored in ADR 0141, and the grouped writing style for `problem`, writing guidance, and the annotation style for `references` were added in ADR 0142. Use the four fields `problem` / `method` / `result` / `limitations`. These four fields are a writing recommendation, not a hard schema constraint, and divide responsibilities as follows.

Trigger timing (ADR 0143): this structure is not reserved only for new files or dedicated batch migrations. From now on, whenever a `papers/` model file is edited for any reason, even just to check a citation, fix one field, or fix a bug, its `description` should be migrated to this section's four-field list structure and its top-level `references` migrated to the `{citation, description}` form below in the same pass, with no need to wait, and no skipping it on the grounds that "this task's scope does not include migration," unless the user explicitly states in the specific task that migration is out of scope.

- `problem`: state the scientific question the model answers in one sentence, then list, grouped by mechanism, why a single source publication cannot answer it, that is, where each group of mechanisms individually stops short. Each mechanism's gap should preferentially draw on the limitation the source publication itself acknowledges rather than the modeler's own guess; verifying each original limitation individually takes time, and when there is not yet capacity to verify, the field can be left blank for a later researcher to fill in, since this is not required of an AI. What specific mechanism or data each source publication contributes belongs in that entry's `description` under the top-level `references`, per the "optional annotation style for references" section below, not in `problem`, so that `problem` stays limited to mechanism-level judgments and is not lengthened by per-citation annotation. Describe it in language a non-specialist reader can follow, dropping framework-internal notation such as T1/T2/T3/T4, K×4, or NSGA-II, and do not describe the debugging or optimization process itself, such as whether the search scale was large enough or whether an earlier statement turned out on review to be coincidental; `problem` describes the scientific question the model answers, not the model's development or debugging history, which belongs in `metadata.todo`, `metadata.log`, or the `history/` directory (see the "Debug History Versioning" section) and does not belong in `description`; see the "Deliverables for the Final Reader Carry No Process Trace" section of the global `~/.claude/CLAUDE.md`.
- `method`: does not list the source publications themselves, only explains how the mechanisms listed in `problem` are combined, that is, which decision variable or resource-competition pathway they share and specifically how joint simulation or joint optimization connects them; see the "LM's core methodology" section at the top of this document. The field has no fixed length; list entries one by one when there are multiple coupling relationships.
- `result`: merges the former `result` and `conclusion`, describing the simulation output or Pareto front. State the conclusion itself in one sentence for each result, then add the detail that supports it; the most important information this field should convey is what combining multiple source publications actually changed and what it left unchanged, so do not mix the conclusion and its derivation into the same sentence and leave the reader to find the conclusion buried among numbers. Either a single solution or multiple solutions is fine, and length varies naturally with the complexity of the result; as with `problem`, do not describe the debugging process, only the final simulation or optimization result itself.
- `limitations`: retained to give future contributors a direction for improvement; this covers the model's own limitations, a different layer from the "limitations of each individual source publication" covered in `problem`, and the two should not be mixed together.
- Removed: `brief`, `need`, `simulation`, `optimization`, `conclusion`.

Writing structure recommendation: whenever `problem`, `method`, or `result` contains multiple parallel facts, write them as a `- `-prefixed list item by item rather than crowding them into one continuous paragraph; the list structure itself makes it obvious what each item is claiming and which two items contradict each other, whereas a mixed-together long paragraph is hard to check even when its content is correct. Only write a single continuous sentence when the content really is one unified judgment with no parallel structure. Whenever these four fields need to name a specific publication in running text, use a parenthetical author-year citation at the end of the sentence, for example "a first-order gastric-emptying rate constant used to compute the decline of stomach milk volume over time (Cavell 1981)," rather than a sentence-initial narrative citation or a numbered reference; a sentence-initial narrative citation only fits contexts specifically discussing what a particular author did, whereas these four fields usually state a mechanism or conclusion itself, with the author not the sentence's subject. The global rules on parentheses, dashes, and hard-wrapping apply to every field of `description` as well, per `~/.claude/CLAUDE.md`: parentheses are prohibited except for mathematical grouping, citation format, abbreviation glosses, and enumeration numbering, where citation format means an author-year academic citation convention and does not count as an inserted aside; dashes are never used. When writing multiple lines with `|` in YAML, each list item occupies its own line, but neither the inside of a list item nor a non-list full sentence should be manually wrapped by character width; no matter how long a sentence is, it stays on one line and the renderer handles the wrapping.

Recommended default form for top-level `references` (ADR 0143): each entry in the array can be a plain string, the full citation text itself, or an object `{citation: "full citation text", description: "the specific mechanism or data this publication contributes to this model"}`, and the two forms may be mixed within the same array. `papers/` models should default to the object form with `description`, which is not an optional nicety, since exactly what mechanism or data each source publication contributes is the model's value relative to any single source publication and the most direct basis a reviewer has for checking whether the coupling actually holds, so it should be written out as a matter of course; only when the contribution genuinely cannot yet be pinned down, or the citation is clearly background, for instance citing the LM format specification itself, should the plain-string form be kept. The GUI displays entries with `description` as two columns, the citation on the left and the contribution note on the right; entries without `description` display as a single column. This way the information about what a publication contributes sits right next to the publication itself, `problem` does not need to repeat it, and citations do not need numbering for cross-reference; the GUI's citation list displays in alphabetical order of the citation string itself, unnumbered.

`references/` and other non-paper models: any fields can be used as needed, with no four-field restriction.

Scientific basis, literature interpretation, mechanism equations, time scales, and modeling assumptions also belong inside `description`, but should be split into more specific fields where possible, such as `evidence`, `mechanism`, `time_scale`, `sources`, or `assumption`. Do not use a sibling `metadata.science_note` or a top-level `science_note`, and using a catch-all `science_note` inside `description` is likewise not recommended.

When a literature source can be pinned to a specific variable or equation, prefer writing it into `variables.<name>.reference` or `equations.<name>.reference` so the GUI's variable and equation views can show the basis directly; only background sources for the scenario as a whole, which cannot be clearly assigned, stay in `description.sources`.

Short text can be written as a plain scalar; use `|` when line breaks need to be preserved:

```yaml
description:
  problem: |
    First paragraph.
    Second paragraph.
```

---
