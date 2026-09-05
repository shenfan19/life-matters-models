# models/papers — Paper-Specific Simulation Scenarios

## Purpose

`papers/` holds **formal simulation scenarios corresponding to a specific academic paper** (in progress or published), used to reproduce the paper's numbers, run controlled comparisons, and generate Pareto-front figures.

Each scenario file is calibrated against literature parameters and includes a complete MC and Opt configuration, reproducing the quantitative results reported in the paper.

Unlike `scenarios/` (objectively usable simulations not yet tied to a specific paper) and `test/` (simulator functionality test cases), files under `papers/` serve directly the writing and review of a specific paper.

Every model file welcomes edits, additional parameter sources, or bug fixes from any user.

## Validation requirements

Each scenario must pass the corresponding tier of the validation protocol; see [`test/test_plan.md`](../test/test_plan.md) (tier 1: analytical solutions, tier 2: literature effect sizes, tier 3: feasible-region/Pareto structure), with execution records in [`test/test_report.md`](../test/test_report.md).
