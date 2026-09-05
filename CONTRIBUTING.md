# Contributing Guide

Contributions of YAML models to the Life Matters model library are welcome!

## Ways to contribute

- **Adding a new model**: build a new physiological, disease, or social-dynamics model based on published literature
- **Improving an existing model**: supply a literature source, correct a parameter, add an optimizer configuration
- **Fixing an issue**: fix a problem model carrying a `metadata.todo` marker
- **Reporting an issue**: submit a problem or suggestion on the [Issue Tracker](https://github.com/shenfan19/life-matters-models/issues)

## Model format

All models must conform to the LM format specification:

- Format introduction: [docs/quickstart.md](docs/quickstart.md)
- The complete format specification: [docs/LM_format_1.0.md](docs/LM_format_1.0.md)
- A modeling-practice guide: [docs/authoring/README.md](docs/authoring/README.md)

## Quality requirements

Before submitting, the following must be satisfied:

1. **Complete `description`**: every variable and equation has a `description` field
2. **Complete `reference`**: every value, range, and equation notes a literature source; fill in `TODO:SOURCE` if there is none yet
3. **sim is runnable**: no `type: nosim` item pending in `metadata.todo` (verify locally with `python cli/batch.py --input-dir <directory>`, which requires the accompanying simulation engine)
4. **Filename convention**: `{topic}_{year}_{author}.yaml`, using snake_case
5. **Original rewriting**: text fields such as `description` must rewrite the literature's mechanism and conclusions in the contributor's own words, not translate or transcribe the original text sentence by sentence, and must not paste a screenshot of a paper's figure/table or a complete table — extract only the values needed for modeling, and note the source through `reference`/`locator`

## Contributor License Agreement (CLA)

Submitting a Pull Request means you agree to the terms in [CLA.md](CLA.md), whose main points are: you have the right to submit this content, you agree to its release under CC BY 4.0, and you are responsible for the parameters' accuracy and the literature citations. No signature is required.

## Submission process

1. Fork this repository
2. Place the model in the corresponding subdirectory (`models/references/medical/`, `models/references/social/`, `models/references/environmental/`, `models/references/risk/`, etc.)
3. Run the simulation engine locally to confirm it passes validation (no pending item in `metadata.todo`)
4. Submit a Pull Request, describing the model's literature source and modeling scenario

## Status markers

Whether a model is "publishable" depends only on the `metadata.todo` field, independent of the filename (ADR 0120):

```yaml
metadata:
  todo:
    - type: nosim | noopt | noref | quality | other
      issue: "A one-sentence description of the problem"
      evidence: "The diagnostic basis, so the next pass doesn't need to re-diagnose"
      next: "The suggested next step"
```

A newly created or unvalidated model should first get the corresponding `todo` item added; once the issue is resolved, remove that item, and an empty `todo` counts as passing — **no filename change is needed**. The complete field description is in the "Status markers" section of [docs/authoring/bookkeeping.md](docs/authoring/bookkeeping.md).
