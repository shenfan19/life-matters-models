# ADR 0150 - Changing metadata.ratings's Scale to a 0-1 Continuous Scale, Adding VESO Four-Question Scoring Fields

## Status

Implemented (the specification itself; migrating the ratings in specific model files under `models/papers/s1/` and `models/plan/` is a separate follow-up task, out of this scope)

## Date

2026-08-20

## Background

This repository previously had two independent, differently scaled expressions of model rating:

1. The var/equ/sim/opt columns of the candidate overview table in `models/plan/lm_model_library_plan.md`, a 0-2 three-tier scale scored before modeling, answering "does this direction have evidentiary support, is it worth investing in."
2. The `metadata.ratings` field in `docs/authoring/ratings.md`, a 1-5 five-tier scale scored after modeling, answering "how important, how in-demand, how good is the evidence quality, how confident is the validation for this already-built model."

During a session on 2026-08-19/20, while looking for a maternal-infant case as a substitute for a degenerate Pareto front in the S1 paper, an "external literature verification" was additionally done for four new candidates and one existing case (CKD), checking whether a published combined quantitative study existed and whether its conclusion was comparable; this kind of judgment is essentially answering the `validation_confidence` field `ratings.md` already had, but it was scored verbally as "high/medium/low" at the time, not aligned to the scale itself. On review it was further noticed that the candidate table's var/equ/sim/opt was in fact deliberately compressed to a 0-2 scale (three tiers) while `ratings.md` uses 1-5 (five tiers), an inconsistency in scale, and that the candidate table's four letters V/E/S/O share the same four-layer breakdown (Variables/Equations/Simulation/Optimizer) as the VESO debug-attribution framework defined in `life-matters-home/process/veso_debug_checklist.md`, two applications of the same core framework, not a coincidental reuse of the same name.

When discussing the rating scale itself, the user raised two design requirements:

- **N out of however many does not self-explain its ceiling**: whether 0-2 or 1-5, an isolated number (such as "2" or "4") cannot state on its own what a perfect score is; a reader must first confirm the scale's definition to know whether this number is near the top or barely passing. 0-1 naturally carries its own ceiling, so "0.7" needs no extra context.
- **The cognitive load of scoring should be low, but not at the cost of unexplainable precision**: a two-tier scheme was initially discussed ("AI uses 1-5 internally, displays 0-2 to the user"), which the user rejected, on the grounds that introducing two parallel numbering systems adds cognitive load and risks the two records drifting out of sync over time; the final decision was a single, continuous 0-1 scale, allowing arbitrary precision (not forced onto discrete tiers), but with each field still anchored at five reference points (0/0.25/0.5/0.75/1, mapping one-to-one to the original 1-5 five tiers), requiring that a deviation from an anchor be explained in the reason text relative to which anchor and why, to prevent continuous precision from turning into unexplainable, spurious precision.

## Decision

### 1. Change every `metadata.ratings` field's scale from a 1-5 integer to a 0-1 continuous decimal

Five anchors at 0/0.25/0.5/0.75/1, with the semantic definitions carried over directly from the original 1-5 five tiers, only the numbers converted, not redefined. Any precision between anchors is allowed, and the reason text must state the basis for the deviation relative to the nearest anchor. A reader-facing interface may optionally display this as five stars (star count equals the score times 5, with a half star corresponding to 0.1 precision), and the underlying value's precision is unaffected by this display conversion.

### 2. Reorganize fields under a technical (VESO) versus non-technical (discretionary) split, streamlining from the original six shared fields plus three type-specific fields to four shared technical fields plus three non-technical fields

In further discussion the same day, the user proposed that the evaluation criteria essentially split into two kinds: VESO (model completeness) judgments have an objective anchor to check against, and are technical; importance, confidence, and novelty have no objective checklist and are essentially discretionary, non-technical. Fields were regrouped and merged accordingly:

- **Technical (VESO, four shared fields)**: `variable` (absorbing the former `evidence_quality`), `equation`, `simulation`, `optimization` (that is, whether the tradeoff holds, absorbing the candidate table's var/equ/sim/opt four questions and the originally designed `o_tradeoff_predicted`), cross-referenced explicitly against the debug-purpose VESO framework (`veso_debug_checklist.md`) as two applications of the same framework, without redefining the standard twice. `optimization`'s number can be overwritten and updated as modeling progresses, but the qualitative prediction text already written into `metadata.description.method` must not be quietly rewritten just because numeric results came in; iterating the number and freezing the predicted text are two different matters, and the originally designed scheme of "freeze `o_tradeoff_predicted` after scoring, and open a separate `validation_confidence` for the reading after validation" was rejected in this round of discussion, because it conflated "the number cannot change" with "the predicted text cannot be quietly rewritten," artificially inflating the field count.
- **Non-technical (three fields)**: `importance` (merging the five former "is it worth doing" judgments, `topic_importance`/`framework_demand`/`paper_value`/`popularity`/`social_value`, into one field, with the one-sentence reason naming which kind of value is driving the score, without requiring every subtype to be listed), `innovation` (the existing field unchanged except for the 0-1 scale, judging novelty rather than importance, an independent axis kept as its own field rather than folded into `importance`), `confidence` (renamed from the former `validation_confidence`, with the judgment basis unchanged; its difference from `optimization` is that the former judges whether the measured result matches the literature while the latter judges whether the trade-off's structure itself holds; the rename reason is that "validation" is no longer the only context in this field system, and simply calling it `confidence` is shorter and fits the "non-technical, discretionary" positioning better).

`ratings.md`'s scope now adds `models/plan/` (previously only `papers/`, `scenarios/`, `references/`), so candidate-material scoring and built-model scoring share the same fields and scale.

### 3. External literature verification's output formally lands on `confidence`, and agent documents no longer define their own standard

The "External Literature Verification" section of `lm-modeling-design.md` used to describe credibility verbally as "high/medium/low," and now scores directly against `ratings.md`'s current `confidence` scale with a reason attached; the parts of `lm-paper-case-writer.md` touching on scoring or citation ratings likewise switch to reading `ratings.md` rather than copying the scale definition into their own document. Going forward, adjusting the scoring standard only requires changing `ratings.md` in one place, and an agent automatically reads the new version the next time it runs.

### 4. This change's scope does not migrate existing ratings retroactively

The 1-5 ratings already written into existing model files under `models/papers/s2/`, `s3/`, `s4/`, `models/scenarios/`, and `models/references/` are not migrated this time; they are kept as-is and read under the definition in effect for each file at the time, left for a separate follow-up task. The historical 0-2 ratings already written in `lm_model_library_plan.md`'s candidate overview table are likewise not rewritten, and are read under that table's own header-labeled old scale. `docs/authoring/ratings.md`, like `LM_format_1.0.md`, is still an unpublished Draft (per ADR 0144's test, with no frozen version and no external consumer depending on the old scale), so this adjustment does not constitute a breaking change to an already-published contract.

## Result

- `docs/authoring/ratings.md`: the rating scale changed to 0-1 with five anchors; fields reorganized into a technical group (`variable`/`equation`/`simulation`/`optimization`, four shared) and a non-technical group (`importance`/`innovation`/`confidence`, with `confidence` renamed from `validation_confidence`), merging and streamlining the original six shared fields plus three type-specific fields into seven fields total, four shared technical plus three non-technical; scope extended to `models/plan/`.
- `agents/lm-modeling-design.md`: "External Literature Verification" now scores directly against `ratings.md`'s `confidence` definition; the "Qualitative Prediction of Pareto Structure" section adds a requirement to score `optimization`, with the number updatable but the predicted text itself never to be quietly rewritten.
- `agents/lm-paper-case-writer.md`: the model-rating snapshot in the `## Plan` subsection and the "introduction" step both now reference `ratings.md`'s current definition (four technical items plus three non-technical items).
- `life-matters-home/process/veso_debug_checklist.md`: a one-sentence cross-reference to `ratings.md` added at the top, noting the VESO framework's dual use.
- This change does not modify the `metadata.ratings` values of any specific model file; migrating the actual ratings for the S1 paper's models and the models under `models/plan/` is carried out as a separate follow-up task.
