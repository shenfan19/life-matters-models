# Model Ratings

> This file is the specification for the `metadata.ratings` field of YAML models in the LM framework.
> Scope: `models/papers/`, `models/scenarios/`, `models/references/`, `models/plan/`.

---

## 1. Overview

`ratings` is an optional block under `metadata` for embedding a structured assessment of a model inside the model file itself.
A rating does not replace the prose in `description`; the two are complementary, with the prose explaining what the model is and the rating answering how much it is worth.

Position: `metadata.ratings`, placed after `description` and before `tags`.

The evaluation criteria fall into two categories, and the fields are organized accordingly:

- **Technical (VESO)**: `variable`/`equation`/`simulation`/`optimization`, answering whether verifiable evidence or support exists at this layer, corresponding to the model's completeness itself, with a relatively objective anchor to check against, such as whether a parameter has a literature source, whether an equation is a recognized form, whether the simulation matches independent data, and whether real trade-off evidence exists. These four layers are the same framework as the V/E/S/O four-layer breakdown used for debug attribution in `life-matters-home/process/veso_debug_checklist.md`, applied in two different ways; see ADR 0150 for the design process.
- **Non-technical**: `importance` (merging the previously scattered judgments of whether something is worth doing, including a topic's real-world importance, demand within the framework, contribution to a paper, academic generality, and value for social discussion), `innovation` (novelty), and `confidence` (overall credibility). These have no checkable objective list and are essentially discretionary, a different mode of judgment from the VESO four layers.

```yaml
metadata:
  name: ...
  description:
    brief: ...
  ratings:
    variable: 0.75 - explanation               # technical (VESO), shared field
    equation: 1.0 - explanation                # technical (VESO), shared field
    simulation: 0.75 - explanation             # technical (VESO), shared field
    optimization: 0.75 - explanation           # technical (VESO), shared field, i.e. whether the tradeoff holds
    importance: 0.75 - explanation             # non-technical, shared field
    innovation: 0.75 - explanation             # non-technical, papers/ and scenarios/
    confidence: 0.5 - explanation              # non-technical, shared field, requires the simulation to have been run and validated before scoring
  tags: [...]
```

---

## 2. Rating Scale (0-1, Five Anchors)

| Anchor | Meaning |
|------|------|
| **1.0** | Highest: sufficient, significant, distinctive |
| **0.75** | High: good, with minor flaws or limitations |
| **0.5** | Medium: adequate, with a clear weakness or something not yet refined |
| **0.25** | Low: supporting or derivative, of limited independent value |
| **0.0** | Lowest: pure placeholder, heavily estimated, or used only for testing |

Format: `field: N - a one-sentence explanation`

```yaml
variable: 0.5 - Placebo effect not modeled, microbiome stability is an aggregate proxy, limited ecological validity
```

The score and its explanation are separated by ` - ` (space, hyphen, space). Keep the explanation to about 30 words, focused on the core basis for the score.

Precision: any value between two anchors is allowed, such as 0.6 or 0.4, and is not forced to land exactly on one of the five anchors, since the rating reflects subjective judgment rather than a measurement, and the precision it warrants is only enough to explain in one sentence why it is higher or lower than a given anchor, not spurious precision to several decimal places. By default, land directly on the nearest anchor; only use a non-anchor value when there is a clear reason to shift in a given direction, and the explanation must state which anchor it is above or below and why, not give a bare number without that relative positioning.

Display: a reader-facing interface may convert the 0-1 value into a five-star display, with the star count equal to the score times 5 (for example 0.7 becomes 3.5 stars), and a half star corresponding to an increment of 0.1; the underlying stored value's precision is unaffected by this display conversion.

---

## 3. Technical Fields (VESO, Shared, Model Completeness)

The following four fields apply to any model under papers/, scenarios/, references/, or plan/, and share the same V/E/S/O four-layer breakdown as `veso_debug_checklist.md`. `docs/authoring/README.md` and the modeling agents read this section's definitions directly when scoring these four fields, rather than defining a separate standard in their own documents.

### `variable`, Variable Evidence (V)

The quality of the objective basis behind the model's parameter and variable values, merging in the scope of what used to be a separate `evidence_quality` judgment, that is, evidence quality across all parameters and mechanisms; the two have been combined into one field.

| Anchor | Standard |
|------|------|
| **1.0** | Every key parameter has a recognized numeric source, such as high-quality RCT, meta-analysis, or a large cohort, directly traceable to the literature |
| **0.75** | Main parameters are well supported by literature, with a few coefficients reasonably estimated |
| **0.5** | A historically derived or self-built equation anchored to a literature point estimate; parameter sources are mixed, some from literature and some reasonably derived |
| **0.25** | Most parameters are estimated or based on rules of thumb, with weak literature support (includes `_noref` models) |
| **0.0** | Pure fabrication with no anchor, or a TODO placeholder |

Note: each parameter tagged `TODO:SOURCE` drops the score by 0.25 directly; a model whose filename contains `_noref` has `variable` at most 0.5; a TODO-stub model has `variable` equal to 0.0.

### `equation`, Equation Support (E)

| Anchor | Standard |
|------|------|
| **1.0** | A widely cited, independently reproduced standard equation form (at the level of Lemaire/FJK/DLNM) |
| **0.75** | A clear literature equation exists, but reproduction is limited in scope or has not been cross-validated by an independent research group |
| **0.5** | A self-built equation anchored to a literature point estimate, or a simplification of part of a recognized standard model's sub-mechanism |
| **0.25** | An equation structure exists but is highly complex and has not been independently reproduced |
| **0.0** | Purely narrative and self-built, with no literature basis |

### `simulation`, Simulation Capability (S)

| Anchor | Standard |
|------|------|
| **1.0** | A simulation already validated point by point against real data or an independent literature trajectory |
| **0.75** | Produces a reasonable time series, with direction and magnitude broadly matching expectations, not yet validated point by point |
| **0.5** | Simulation configuration is complete but has not been run and validated, or runs successfully but lacks an independent data comparison |
| **0.25** | Only qualitative inference is possible, not actually validated |
| **0.0** | No simulation configuration, or never run |

### `optimization`, Optimization Trade-off (O, Whether the Tradeoff Holds)

Judges whether the model's decision variables constitute a real trade-off against its objectives, not "several metrics all want to improve" but "improving one truly costs another." At the candidate stage this is a directional judgment based on equation derivation; once the model is built and runs validated, it is reassessed together with the numeric results, and the number itself can be updated as modeling progresses (overwriting the old value, with version control keeping the history; see Section 6).

| Anchor | Standard |
|------|------|
| **1.0** | Specific, independently quantified research gives concrete two-directional real-harm evidence (a genuine non-degenerate trade-off backed by numeric evidence) |
| **0.75** | RCT/meta-analysis-level evidence supports a real trade-off, but the evidence chain has to be pieced together across publications |
| **0.5** | The direction holds, with precedent support but no numeric validation |
| **0.25** | Weak evidence, direction uncertain, or numeric validation shows signs of degeneration |
| **0.0** | No trade-off, or structurally inapplicable (confirmed by tracing that the objectives move in the same direction with no trade-off, or the model by design has no decision variable) |

Relationship to the qualitative prediction text: the `optimization` number can be overwritten and updated, but the qualitative prediction text written into `metadata.description.method` once the equations are finalized must not be quietly rewritten just because numeric results came in later. The number is allowed to iterate; prediction text that has already been committed must be kept as written, with any discrepancy recorded separately. These are two different things, and the latter continues the principle already established in the "Qualitative Prediction of Pareto Structure" section of `lm-modeling-design.md`.

---

## 4. Non-Technical Fields (Discretionary)

The following three fields have no checkable objective list and are essentially subjective, holistic judgments.

### `importance`

Merges what used to be several separate "is it worth doing" judgments: a topic's real-world importance (the size of the affected population, its standing as basic science, the level of social attention), demand within the framework (the expected strength of being imported or referenced by other models), contribution to a paper, academic generality, and social, ethical, or historical discussion value. These are all specific types within "there are several kinds of value" and are no longer split into separate fields; the one-sentence reason only needs to name which type or types are mainly driving the score this time, without being required to cover all of them every time.

| Anchor | Standard |
|------|------|
| **1.0** | A global issue affecting billions or a core mechanism of basic science; or a core framework building block that multiple upper-level models depend on; or a paper's flagship case; or a textbook-level general benchmark; or touches a universal ethical or major historical lesson |
| **0.75** | An important specialty or social issue affecting hundreds of millions; or depended on by multiple scenarios or papers; or one of a paper's core cases |
| **0.5** | A specific population's or discipline's issue; or moderate-frequency use in a specific subfield |
| **0.25** | A niche topic with a limited audience; or low-frequency citation, a purely supplementary appendix |
| **0.0** | Pure placeholder or test, with no real-world counterpart; or purely for entertainment display with no substantive discussion value |

### `innovation` (papers/ and scenarios/ Only)

Methodological or scientific novelty, the distinctive contribution relative to existing literature, a separate matter from how important the topic is. A topic can be important while the modeling approach has nothing new, or an obscure topic can carry a very novel modeling angle. Innovation is not the same as complexity; a simple model introducing a mechanism or optimization angle not previously tried on this problem scores higher than a complex model that repeats a known path. Test: could a reviewer find an almost identical model already published in the literature?

| Anchor | Standard |
|------|------|
| **1.0** | Distinctively original, hard for a reviewer to find a precedent for comparison |
| **0.75** | High-quality precedent exists, but the combination or application context is a new contribution |
| **0.5** | Partially novel, with the core mechanism following a known path |
| **0.25** | Limited incremental contribution |
| **0.0** | Ample precedent already exists, with a negligible incremental contribution |

Candidate material under `models/plan/` leaves `innovation` blank until the candidate is adopted and it is settled whether it goes into `papers/` or `scenarios/`.

### `confidence`

Assesses whether the model's simulation or optimization output, once actually measured, matches an independent literature target; this is a one-time, holistic confidence judgment, a different mode of judgment from fields like `variable`/`equation` that can be checked item by item against a literature source, and the two should not be conflated. Write only a 0-1 number plus a one-sentence reason, without expanding into sub-fields; its update timing follows the file's own `metadata.updated` rather than maintaining a separate date.

```yaml
metadata:
  ratings:
    confidence: 0.75 - one-sentence basis (e.g. "The BP reduction, once decomposed, falls within the literature range; the UA increase does not meet the bar due to scenario confounding")
```

| Anchor | Meaning |
| --- | --- |
| **0.0** | Not yet validated: only a static parameter declaration exists, `--sim`/`--opt` has not been run successfully, or a known blocking bug exists |
| **0.25** | Runs but does not match the literature: produces a simulation trajectory normally, but it has not been compared against an independent literature target, or the comparison shows the wrong direction (a coefficient suspected to come from a different source, unit or timescale conversion not tracked, etc.) |
| **0.5** | Partially benchmarked: partially matches a literature target, or matches but the scenario or sample has a clear limitation, such as confounders not separated out or limited representativeness from a single sample |
| **0.75** | Broadly matches: the core validation point falls within the literature target's range, with deviation at an acceptable magnitude; or the declared evidence effect size itself comes from a high-quality independent source and the loader encodes and displays it correctly (note in the reason text when this is not an independent prediction) |
| **1.0** | Passes independent predictive testing: a leave-one-out prediction using independent data points not involved in fitting matches what the literature reports, or multiple independent sources cross-confirm consistently |

The cross-discipline validation case summary and per-model detailed basis are in `models/test_validation/validation_report.md`; when a paper cites a model, prefer models with `confidence >= 0.75` as core cases, while models at 0.5 or below are suitable for demonstrating methodology or framework capability but need their limitations stated in the paper's own text and should not support a strong quantitative conclusion.

For a candidate or newly drafted model, the credibility judgment produced by the "External Literature Verification" section of `lm-modeling-design.md` is exactly the score for this field; read this section's definitions directly when scoring, rather than defining a separate colloquial high/medium/low standard in the agent documents.

---

## 5. Semantic Notes

### Cross-Type Consistency of `variable`

For historical scenarios, the evidence is historical literature and physical or physiological literature; for reference models, it is the original studies cited directly; for paper models, it is RCT and meta-analysis sources. The definition is unified, and it adapts naturally to context.

### The Difference Between `optimization` and `confidence`

Both concern whether the model's trade-off holds up, but from different angles: `optimization` judges whether the trade-off's structure holds, that is, whether the decision variable truly has an opposite-signed effect on the objectives; `confidence` judges whether the measured result matches independent literature, that is, whether the numbers fall in a reasonable range and are corroborated by external evidence. A model can score high on `optimization` (the trade-off is structurally real) while scoring only medium on `confidence` (numeric validation turned up a detail problem, such as a missing saturation mechanism); the two numbers do not determine each other.

---

## 6. Rating Update Rules

- After each round of parameter refinement, such as filling in a `TODO:SOURCE`, update `variable` accordingly.
- After a model's positioning changes, such as being promoted to a flagship case or demoted to an appendix, update `importance` accordingly.
- `optimization` and `confidence` are updated with each revalidation, overwriting the old value; but the qualitative prediction text already written into `metadata.description.method` must not be quietly rewritten just because numeric results came in, per the note under the `optimization` entry in Section 3.
- A rating is the modeler's subjective judgment, and version control keeps its history; do not delete an old rating's comment, just overwrite the value.
- Ratings do not enter the engine's computation and are for the modeler's and collaborators' reference only.
- Whenever an agent document outside this file (`lm-modeling-design.md`, `lm-modeling-inspiration.md`, `lm-paper-case-writer.md`, etc.) touches on scoring, it reads this file's current definitions rather than copying or separately defining a scale in its own document; a scoring-standard change is made in this one file only.

---

## 7. Examples

### Full papers/ Example

```yaml
metadata:
  name: a5_hypertension_gout
  description:
    brief: The clinical conflict of HCTZ treating hypertension while raising uric acid, a T2+T4 chronopharmacology optimization.
  ratings:
    importance: 1.0 - Hypertension plus gout is one of the most common drug conflicts, affecting a large patient population, a paper's core case
    variable: 0.75 - Supported by MAPEC RCT evidence, with the uric acid circadian-rhythm coefficient still to be refined
    equation: 0.75 - The chronopharmacology equations have a clear literature basis, with the combined form original to this paper
    innovation: 0.75 - A chronopharmacology framing; high-quality precedent exists, but the multi-objective combined form is a new contribution
  tags: [paper2, hypertension, gout, ...]
```

### Full scenarios/ Example

```yaml
metadata:
  name: ad1847_hu_semmelweis
  description:
    brief: A simulation scenario of Semmelweis's handwashing advocacy (1847-1865).
  ratings:
    importance: 0.75 - Medical history's most famous case of resistance to new knowledge, drawing high attention in medical and science-policy circles, with direct relevance to evidence-based-medicine policy discussion
    variable: 0.75 - Rogers's diffusion theory and Carter's biography provide a good literature basis
    innovation: 1.0 - The first modeling of scientific-innovation acceptance dynamics as an optimizable problem
  tags: [historical, semmelweis, ...]
```

### Full plan/ Example (Including the VESO Four Questions, Added 2026-08-20)

```yaml
metadata:
  name: aircrew_circadian_sim
  description:
    brief: The trade-off between operating cost and on-duty fatigue for aircrew scheduling across time zones.
  ratings:
    importance: 0.75 - Affects flight crews worldwide, a real contest between safety and operating efficiency; currently an isolated candidate with no expectation yet of being imported
    variable: 0.75 - The FJK model's core parameters are published and cite the Rea 2022 calibrated version; light-therapy intensity and half-width are engineering calibrations
    equation: 1.0 - The FJK third-order ODE is a widely cited standard form reproduced continuously since 1999
    simulation: 0.75 - The resynchronization timescale independently matches Serkh & Forger, but has not been compared point by point against data
    optimization: 0.75 - The competing relationship between crew-scheduling operations and fatigue is literature-supported, and the front's magnitude independently matches after numeric validation
    confidence: 0.75 - The counterintuitive finding about light-therapy timing was confirmed by passing independent predictive testing, which alone would warrant 1.0, but the missing sleep-debt mechanism and insufficient validation budget pull the overall score down
  tags: [aircrew, circadian, fatigue, ...]
```

### Full references/ Example

```yaml
metadata:
  name: ckd_protein_muscle
  description:
    brief: A dynamic model of protein intake versus muscle preservation in CKD.
  ratings:
    importance: 0.75 - CKD affects hundreds of millions of patients worldwide, and nutrition management is a core clinical issue; already imported by the A4 paper series' scenarios, with a solid standing in the nephrology field
    variable: 0.75 - Supported by KDIGO guidelines and several RCTs, with a few coefficients estimated
  tags: [nephrology, ckd, ...]
```
