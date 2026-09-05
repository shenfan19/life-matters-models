# models/test_fixtures/valid — Simulator functionality test cases (correctness)

## Purpose

`test_fixtures/valid/` holds **structurally valid, minimal examples used to test and explain each simulator feature**, each file focused on one LM format feature (such as imports composition, MC distributions, each tier of K x 4 optimization, equation conditions, phased schedules, etc.). This is the counterpart to `test_fixtures/invalid/` (deliberately broken, used to verify the error-detection mechanism, see its README).

These files do not represent a real clinical or social scenario; their main purposes are:
- verifying that the corresponding simulation-engine/optimizer feature works correctly
- serving as a minimal, readable example of that feature, for developers and model authors to reference

Unlike `papers/` (paper scenarios), `scenarios/` (objectively usable simulations), and `references/` (base submodel components), `test_fixtures/valid/` needs no literature-parameter calibration.

Every model file welcomes edits, additional test cases, or bug fixes from any user.
