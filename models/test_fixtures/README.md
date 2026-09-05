# models/test_fixtures — A YAML fixture library for engine verification

## This directory tests verify, not validate

Several directories in this repository have easily confused names; here is what each one actually tests:

- **`models/test_fixtures/` (this directory)**: a set of YAML fixtures (`valid/` structurally valid, `invalid/` deliberately broken), which are not test code themselves but supply input data for tests elsewhere, checking whether the engine can correctly load and run a valid model, and whether it reliably reports errors on a broken one. This tests **whether the engine code itself is written correctly** (verify), and has nothing to do with any model's scientific credibility; `valid/README.md` and `invalid/README.md` already note that "neither represents a real clinical or social scenario, and neither needs literature-parameter calibration."
- **`test_verification/` (the root of the life-matters-reference-engine repository)**: the actual assertion-running pytest code, which reads the fixtures in this directory as input and asserts that the load/run results match expectations. The methodology and results are in `test_verification/verification_report.md`, **the project's only verification-result report**, not duplicated here.
- **`models/test_validation/`**: an entirely different matter, literature benchmarking (whether a model's simulation results fall within the range reported by published literature), optimization plausibility, and so on, testing **whether a model represents the real world** (validate), unrelated to the fixture content in this directory. Note this directory once shared the same name: on the morning of 2026-07-21, ADR 0135 first split the then-mismatched `models/test_validation/` (which held this directory's fixtures) into this directory `models/test_fixtures/` plus a newly created pure-report directory `models/validation/`; that afternoon, ADR 0136 renamed `models/validation/` back to `models/test_validation/` (by which point its content was already the pure validation_report.md, no longer mismatched), which is not the same directory as this one, just a name reused in sequence. See `models/test_validation/validation_report.md`.

One-sentence distinction: **this directory (plus `test_verification/`) asks "was this line of engine code written correctly," while `models/test_validation/` asks "is this model/this simulation result credible."**

## Directory contents

[`fixture_catalog.md`](fixture_catalog.md): an item-by-item explanation of every YAML fixture under `valid` and `invalid`, grouped by functional area, covering what each tests and why it is tested separately.

- **`valid/`** — minimal, functional examples that are structurally valid and load and run normally, each file focused on one LM format feature (imports composition, MC distributions, each K x 4 optimization tier, equation conditions, evidence subtypes, etc.), with filenames uniformly starting with `test_valid_`. See `valid/README.md` for its purpose.
- **`invalid/`** — deliberately broken fixtures, used to verify the engine's **error-detection mechanism**: loading/validation must fail, and the failure reason must be visible through a normal call path such as `ReferenceEngine.load_models()` (not just logged), with filenames uniformly starting with `test_invalid_`. See `invalid/README.md` for its purpose; the corresponding pytest assertions are in `test_verification/errors/`.
