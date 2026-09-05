# models/scenarios — Objectively Usable Simulation Scenario Library

## Purpose

`scenarios/` holds **objectively usable simulation scenarios not yet tied to a specific paper**.

Unlike `papers/` (serving a specific paper's writing) and `test_fixtures/` (testing the simulator's functionality), `scenarios/` is an open candidate library of scenarios: each scenario can run independently, describing the dynamics of a real or historical person or group under specific conditions, available for future papers, teaching, or case studies to draw from.

## Directory structure

```
medical/   Medical and health-related scenarios (disease management, nutrition, exercise, etc.)
social/    Social/historical/humanities-related scenarios (historical figures, group events, etc.)
```

## Content reliability

The scenario files in this directory were generated with substantial AI assistance, and most have not yet been verified by a domain expert; the degree of verification is given by each file's `metadata.ratings.confidence` field (a continuous 0-1 scale, defined in [`docs/authoring/ratings.md`](../../docs/authoring/ratings.md)); the full reliability boundary is stated in the "Content Reliability Statement" section of the repository root README.

Every model file welcomes edits, additional parameter sources, or bug fixes from any user.
