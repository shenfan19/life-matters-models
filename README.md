<img src="icon.svg" width="48" height="48" alt="Life Matters icon" />

# Life Matters · Model Library

LM builds systems-science models grounded in modern life-science research findings, running multi-objective optimization over decisions in life and living; when optimization has no single optimal solution, the result is presented as a Pareto front, handing the full set of better possible combinations to the user for evaluation and reference. This repository is the content library for the Life Matters project and the foundation of the whole project, publishing the LM format specification together with models based on published literature. Simulating and optimizing a model requires the companion [life-matters-reference-engine](https://github.com/shenfan19/life-matters-reference-engine), whose online demo needs no installation and is the fastest way to try a model: [http://137.184.220.139](http://137.184.220.139).

---

## Disclaimer

The historical and medical scenarios in this project are based on published academic literature and are intended solely for health-decision education. None of the simulated content represents a moral judgment of any historical figure; historical data has been simplified and does not constitute medical advice; simulation results are model projections, not a reconstruction of historical fact.

---

## What this is

This repository is the foundation repository of the Life Matters project, holding the LM format specification itself along with physiological, nutritional, disease, and social-dynamics models based on published literature.  
Each model is a YAML file written in the LM format, directly runnable and optimizable by the LM Reference Engine.

LM format is an open YAML format standard, similar to SBML / CellML, but focused on:
- **Individual-scale** health and behavioral dynamics (minutes to years)
- **Behavioral-intervention scheduling** (diet, exercise, medication timing)
- **Multi-objective Pareto optimization** (searching for an optimal intervention plan)

Core capabilities:

1. Converting statistical conclusions from medical/social-science literature (OR, HR, Cohen's d, etc.) into a runnable YAML dynamics model
2. Running models at different scales (minute-hour-day-year) simultaneously within a unified framework
3. Multi-objective Pareto optimization of a behavioral-intervention plan (Regimen)
4. Putting parameters from multiple publications into the same framework, testing whether they are mutually self-consistent (Simulation-as-Validation)

An LM file is made of four top-level mechanisms, read together as V.E.S.O.: `variables` (transferable numerical evidence), `equations` (wiring the evidence into dynamics that evolve over time), `simulation` (running out a trajectory), and `optimizer` (searching the decision space for tradeoffs). Four questions decide whether a candidate topic falls within this scope; see the Inclusion Test in the Scope section of [`docs/LM_format_1.0.md`](docs/LM_format_1.0.md).

The models in this repository do not reproduce a single study's conclusion paper by paper; instead, they place mechanisms each independently validated by separate publications into the same model, letting mechanisms that would otherwise never have known of each other genuinely interact, revealing tradeoffs invisible within any single paper's own modeling scope. The methodology for deciding which mechanisms a model should include, and which it should exclude, is detailed in the opening section "LM's core methodology: coupling, not stacking" of [`docs/authoring/methodology.md`](docs/authoring/methodology.md).

---

## Content Reliability Statement

This repository's positioning is close to a computable scientific reference library for the AI era: the model content is generated with substantial AI participation, an unavoidable reality of this era, and making this kind of content checkable and correctable is part of the reason this repository exists. Two boundaries are stated explicitly for this purpose:

- **The author's own responsibility**: the LM format specification, [`docs/LM_format_1.0.md`](docs/LM_format_1.0.md); the accompanying simulation and optimization engine, see [life-matters-reference-engine](https://github.com/shenfan19/life-matters-reference-engine); and a small number of core exemplar models already marked "Strong validation," with the criterion `confidence` no lower than 0.75; the complete list is in the inclusion-criteria table in [`docs/authoring/methodology.md`](docs/authoring/methodology.md) and in [`models/test_validation/validation_report.md`](models/test_validation/validation_report.md).
- **The rest of the model library**: most model files are generated and given a first-pass review with AI assistance, and **have not yet been checked by a domain expert**; they are for methodology demonstration and testing reference only, and do not constitute a clinical or scientific conclusion. Each model's `metadata.ratings.confidence` field marks its current level of verification, a continuous 0-1 scale, defined in [`docs/authoring/ratings.md`](docs/authoring/ratings.md); the inclusion-criteria table and the validation report honestly record which disciplines and models have been validated and which are still pending assessment — judge credibility from these, and do not assume a published model has already been verified.

If you are an expert in a relevant field and find a parameter, mechanism, or conclusion wrong in some model, please submit an issue or PR pointing it out — this is exactly why the models are published in an open, checkable YAML format rather than locked inside a private tool.

---

## Repository structure

```
models/
  references/     Literature-based reference component models
    medical/        Physiology, nutrition, disease, pharmacology
    social/         Economics, conflict, psychology, demography
    environmental/  Environmental science
    risk/           Actuarial science and risk
  papers/         Complete scenarios tied to a paper (including optimization results)
  scenarios/      Composed scenarios (under development)
  test_fixtures/  Simulator functionality test cases
  test_validation/ Model validation reports and results
docs/
  LM_format_1.0.md   The complete format specification
  authoring/    A modeling-practice guide, indexed at authoring/README.md (methodology, writing conventions, ratings, regimens/optimizer, etc.)
  quickstart.md Write your first model in 30 minutes
  decisions/    Architecture decision records (ADRs) for the YAML format and model-library structure
```

The specific content, validation status, and contribution conventions for each model directory are in its own README: [`models/references/README.md`](models/references/README.md), [`models/papers/README.md`](models/papers/README.md), [`models/scenarios/README.md`](models/scenarios/README.md).

---

## Quick start

**Read the format specification**: [docs/quickstart.md](docs/quickstart.md) → [docs/LM_format_1.0.md](docs/LM_format_1.0.md) → [docs/authoring/README.md](docs/authoring/README.md)

**Run a model**: requires the accompanying LM Reference Engine, see [life-matters-reference-engine](https://github.com/shenfan19/life-matters-reference-engine); that repository's README also links a live online demo, no local install needed.

**Batch testing** (run from within the life-matters-reference-engine repository, see [cli.md](https://github.com/shenfan19/life-matters-reference-engine/blob/main/docs/cli.md) for detail):

```bash
python cli/batch.py --input-dir models/references
```

---

## Documentation index

| Document | Content |
|------|------|
| [docs/LM_format_1.0.md](docs/LM_format_1.0.md) | The complete LM format specification (the academic edition) |
| [docs/authoring/README.md](docs/authoring/README.md) | An index of the modeling-practice guide (writing conventions, ratings, regimens/optimizer, etc.) |
| [docs/quickstart.md](docs/quickstart.md) | A getting-started guide: write your first model in 30 minutes |
| [docs/authoring/ratings.md](docs/authoring/ratings.md) | The model-quality rating system (the metadata.ratings field explained) |
| [docs/DECISIONS.md](docs/DECISIONS.md) | An index of architecture decisions for the format and library structure |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

---

## Citation

Fan Shen. *Life Matters format: A YAML Specification for Behavioral Intervention Simulation and Optimization in Life Dynamics.* Manuscript submitted for publication. This section will be updated with an arXiv/DOI link once available.

---

## Acknowledgments

This project was developed with AI coding assistance for code generation, automated testing, and documentation.

## License

CC BY 4.0 · Copyright (c) 2026 Fan Shen
