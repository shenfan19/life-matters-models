# 0145 - The LM Modeling Collaboration Agent Four-Step Pipeline and Document Placement

**Date**: 2026-08-15
**Status**: Accepted

---

## Background

LM modeling work used to depend on the user manually working through the whole sequence of finding a direction, drafting a YAML, diagnosing it, and proposing and validating candidate changes, re-explaining the specification and re-explaining the diagnostic dimensions every single time, with the collaboration instructions never accumulating into something reusable. Now that Claude Code's subagent mechanism is available, the conditions exist to fix this collaboration process into a reusable, portable set of documents that an AI assistant can execute through standardized steps, while keeping human verification as the final check.

## Decision

Adopt a four-step pipeline, with the complete instruction documents kept in this repository's `agents/` directory:

- **Step 1** `lm-modeling-inspiration.md`: finding inspiration. Based on the discipline coverage inventory table (`docs/model.md`) or a direction given by the user, does literature research and produces a list of candidate modeling directions, creating no file.
- **Step 2** `lm-modeling-design.md`: drafting and structural changes. Writes a direction or a structural-change requirement into a runnable model YAML draft; drafts always land in `models/temp/` and never touch a formal model file.
- **Step 3.1** `lm-modeling-diagnosis.md`: diagnosis. Runs a comprehensive diagnosis of an LM model, decision-variable boundaries, structural conflicts in the feasible region, physiological-boundary violations in the simulation trajectory, the Pareto front's shape, citation gaps, self-reported known issues, and produces a unified diagnostic table. The model is not required to have an `optimization:` block; a sim-only model can be diagnosed too; this step has no editing capability.
- **Step 3.2** `lm-modeling-advisor.md`: research and execution. Based on the diagnostic table, researches literature support and proposes candidate modeling changes, actually editing a model copy, running life-matters-reference-engine to validate, and iterating on the results. Never edits a formal model file, editing only the `temp_probe/` and `temp_advisor/` copies under the model's own directory.
- **Step 4** `lm-agent-report.md`: the task-report convention. Not an independently triggered agent, but a wrap-up rule the first four documents all follow: once any agent has actually produced something that will be referenced or depended on later, it records the background, task, environment, process, result, reproducible expectation, and follow-on state per a template.

Step 3.1 and 3.2 together form the "adjustment" stage and are typically used together; Step 1 and Step 2 are each independent.

### Document placement: complete instructions live in the models repository; the reference_engine repository holds only a trigger entry point

The complete instructions are maintained uniformly in this repository's `agents/*.md`, not tied to any particular AI tool or framework; the documents are designed to be readable in full and followed by any AI assistant that supports long-context instructions, without depending on any Claude-Code-specific mechanism.

`life-matters-reference-engine/.claude/agents/` holds thin wrapper stubs for the corresponding four documents, each doing exactly one thing, pointing to the same-named document in this repository's `agents/` directory and reading and executing it; if the sibling repository does not exist, it honestly tells the user it cannot proceed. That repository's entire `.claude/` directory is excluded via `.gitignore`, so these stubs never enter version control; the single source of truth for the real content is always this repository's `agents/` directory, and Claude Code is just one of many possible execution environments.

### Engine validation runs by default; a plain-text-only analysis requires explicit instruction

Steps 2/3.1/3.2 call life-matters-reference-engine for validation by default (Step 2 is syntax/runnability validation, Steps 3.1/3.2 are numeric validation). Falling back to plain-text analysis only, reading the YAML, reasoning, and web search, with no computation run, requires the user to state this explicitly in conversation; if the `life-matters-reference-engine` repository does not exist at all in the calling environment (the two repositories must exist as sibling directories), it likewise falls back automatically and states the reason. Step 1 does not touch the engine and only does research.

### Temporary-file convention

Each document creates `temp_probe/` and `temp_advisor/` subdirectories as needed under a model's own directory to hold temporary copies; Step 2's drafts uniformly go into `models/temp/`. A new `**/temp_*/` rule is added to `.gitignore`, excluding any directory at any depth with a `temp_` prefix or suffix, to prevent probe or draft copies from polluting formal model files or being committed by mistake.

### Step 4 reports are kept in the internal repository

The actual task reports Step 4 produces are kept in `life-matters-home/agent_reports/`, not in a public repository, consistent with the precedent of `models/test_validation/validation_report.md`: real usage records accumulate internally first, and once this agent process has a credible track record, a decision is made about which excerpts to make public; no public repository file currently references this directory, and no placeholder file is pre-created.

## Scope of Impact

- New files: `agents/README.md`, `agents/lm-modeling-inspiration.md`, `agents/lm-modeling-design.md`, `agents/lm-modeling-diagnosis.md`, `agents/lm-modeling-advisor.md`, `agents/lm-agent-report.md`.
- A new `**/temp_*/` rule added to `.gitignore`.
- Four local stubs added under `life-matters-reference-engine/.claude/agents/` (not entering version control).
- `life-matters-home/agent_reports/` set up as Step 4's internal report directory, with the first report already recording, retroactively, the Step 3.2 runs actually carried out on 2026-08-15 for the three models ibs_diet/masld_insulin/bergman_glucose.

## Result

- The modeling collaboration process moved from "re-explained verbally every time" to four reusable, portable, standardized documents, with permission boundaries, who can create files, who can edit formal files, who can only diagnose and not change anything, written explicitly into each document rather than relying on an ad hoc convention.
- Engine validation is on by default, ensuring a draft or candidate change has already passed a syntax or numeric check before being handed back to the user, rather than staying at the level of plain-text reasoning.
- Editing permission on formal model files stays tightly scoped: of the four documents, only Step 3.2 can edit a model copy, and it explicitly excludes formal files; whether to actually adopt a change still requires human verification.

## Open Questions

- Whether, and when, excerpts of Step 4 reports need to be made public is not pre-decided by this ADR and is left for later judgment based on the usage track record.
