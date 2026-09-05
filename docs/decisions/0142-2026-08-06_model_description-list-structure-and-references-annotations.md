# ADR 0142 - Changing description to a List Structure, Moving Source-Literature Contribution Notes into references

## Status

Implemented

## Date

2026-08-06

## Background

ADR 0141 set `papers/` model `description` to the four fields `problem/method/result/limitations`, but writing against it in practice exposed two problems.

First, source literature was mixed into `method`'s prose, leaving the reader to pick out from a whole paragraph which papers this model actually used, when that is exactly the information LM most needs to highlight relative to reproducing a single paper, and it deserved a clear place to be listed directly.

Second, when `problem`/`result` were written as a single unbroken long paragraph, several parallel facts, several groups of numbers, and several source publications' contributions were all crowded together, making it hard for a reader to tell which sentence was saying what, and hard to check whether two adjacent sentences contradicted each other. The `result` field in a few demonstration-variant files under `s1/infant_breastfeeding` was a typical case: a single paragraph mixed together a conclusion, its supporting data, the diagnostic process, and the next step, leaving a reader unable to grasp what the paragraph's actual conclusion was after reading it.

## Decision Process

This change converged in three steps; recording them here so a later reader knows why the two intermediate approaches were abandoned and does not need to try them again.

**Step 1, a standalone `citation` field.** Source literature was split out into a new field, placed between `problem` and `method`, formatted as "Author Year: what mechanism or data this publication specifically supplies to this model." After trying this out on every current-version file under `s1/infant_breastfeeding`, it turned out to overlap heavily with the item in `problem` explaining why each source alone cannot answer the question, forcing a reader to cross-reference between the two fields to see which gap a given publication filled, which added reading burden instead of reducing it; the standalone `citation` field approach was abandoned.

**Step 2, a nested sub-item under `problem`.** The literature-contribution note was attached directly under the mechanism item it belonged to inside `problem`, forming a two-level list, an outer level of mechanism gaps and an inner level of the specific literature each mechanism depends on. This solved the cross-referencing problem, but quickly exposed a new one: once a model combined many publications, `problem` itself grew very long, a two-level indented list was still hard to read, and the mechanism-level judgment got buried under specific-literature detail.

**Step 3, moving the contribution note into `metadata.references`.** What a given publication specifically contributes is, in essence, a property of that publication itself, not a property of `problem`, and it should hang off the reference entry it belongs to rather than being copied and stuffed into `problem`. Each entry in `metadata.references` was accordingly expanded from a plain string into an optional `{citation, description}` object, and `problem` kept only the mechanism-grouping and gap-statement layer of judgment, no longer nesting specific literature inside it. The GUI sidebar and exported-report reference box were updated to support a two-column display accordingly, the citation itself on the left and the contribution note on the right.

For citation format, an author-year style (Chicago style) was compared against numbered citations. Numbering is shorter, but the GUI renders plain text with no click-through, so a reader still has to manually count which entry a number refers to; these YAML files are also routinely rewritten iteratively, so any change to the reference list's order requires re-checking every number in the body text with no validation to flag a missed one; and fields such as `variables.<name>.reference` and `formulas.<name>.reference` already uniformly use an author-year format, so numbering would create two citation styles within the same file. Author-year was settled on, with no numbered citations introduced; the year is set in a parenthetical, which falls under the citation-format exemption in the global parenthesis rule.

For a citation's position within a sentence, a sentence-initial narrative form, such as "Cavell (1981) reported that...," was compared against a sentence-final parenthetical form, such as "...(Cavell 1981)." The sentence-initial form lets a reader spot whose publication it is at a glance, but academic citation convention reserves a narrative citation for contexts specifically discussing what a particular author did, whereas this text states the mechanism itself, with the author not the sentence's subject, so a sentence-final parenthetical is the conventional choice; a reader's need to scan by author is already served by `metadata.references`'s full bibliography and does not need repeating in the body text. A sentence-final parenthetical was settled on uniformly.

The GUI's reference box used to display entries with an array-index number, such as `[1]` `[2]`; this change also removed the numbering, switching to ordering by the citation string's own alphabetical order: these citations are uniformly in a "Surname (Year) Title..." format, so sorting directly by string is equivalent to sorting by the first author's surname, and a reader looking up an entry from the body text's author-year parenthetical can locate it by surname more directly than by counting a number, which also avoids the visible run of consecutive numbers giving the false impression that body-text citations cross-reference by number.

## Decision

### 1. `papers/` model description keeps its four fields: `problem / method / result / limitations`

- `problem`: state a topic sentence first, then list, grouped by mechanism, the boundary each group of mechanisms fails to cover on its own, which is the reason the model needs to combine several publications; the boundary description should preferentially draw on a limitation the publication itself acknowledges, and can be left blank when it cannot yet be verified paper by paper, without requiring an AI to complete it. Specific publications are not listed here; that is `metadata.references`'s job.
- `method`: does not list the source publications themselves, only explains how the mechanisms in `problem` are combined, that is, which decision variable or resource-competition pathway they share.
- `result`: state the conclusion itself for each result first, then add supporting detail, focusing on what combining several publications changed and what it left unchanged.
- `limitations`: unchanged, the model's own limitations, a different layer from the per-source limitations in `problem`.

The four fields are a writing recommendation, not a hard schema constraint; non-`papers/` models such as `references/` are not bound by this.

### 2. `metadata.references` adds an optional two-field object form

Each array entry can be a plain string, the full citation text itself, or an object `{citation: "full citation text", description: "the specific mechanism or data this publication contributes to this model"}`, and the two forms can be mixed within the same array. `description` should stay restrained, stating only this publication's role in this specific model, not a summary of the publication.

### 3. Writing-structure recommendation: parallel facts as a list, citations uniformly as a sentence-final parenthetical

Whenever `problem`/`method`/`result` contains several parallel facts, write them item by item as a `- `-prefixed list; write a single continuous sentence only when it truly is one unified judgment. An author-year citation is always placed in a parenthetical at the end of the sentence, never at the start. The global rules on parentheses, dashes, and hard-wrapping apply to every field of `description` as well; when YAML writes multiple lines with a `|` block, each list item occupies its own line, but neither the inside of a list item nor a non-list full sentence should be manually wrapped by character width.

## Impact

- The `metadata.description` and `metadata.references` sections of `docs/model.md` have been updated to the final form.
- `gui/src/components/sim_tab/SimIntroTab.tsx` and `gui/src/components/sim_tab/ReportButton.tsx` now support the object form of `references`, alphabetical ordering, and two-column rendering; the four `locales/engine/*.json` files have gained the corresponding column-header translations.
- The two files with a genuine full reference list, `models/papers/s1/infant_breastfeeding/infant_breastfeeding_sim.yaml` and `infant_breastfeeding_night_tradeoff_demo.yaml`, have been rewritten to the final form; the other variant files in the same directory already use a pointer form referring to these two files and are unaffected.

## Non-Goals

- This does not turn the four fields or `references`'s object form into a hard schema constraint; `references/` models are unaffected.
- This does not introduce numbered citations.
- This does not rewrite the rest of the models under `models/papers/` retroactively in bulk; this ADR only adds the field definitions and writing recommendations, and migrating historical models is separate follow-up work.
