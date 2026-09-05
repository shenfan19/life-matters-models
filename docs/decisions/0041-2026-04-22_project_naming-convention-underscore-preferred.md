# ADR 0041 - Project Naming Convention: Underscores (snake_case) Preferred
**Date**: 2026-04-22
**Status**: Decided

---

## Background

The project currently includes a Python backend (`sim_engine`), frontend code (`sim_gui`, `game`), and Markdown documentation (`docs/game_design.md`). In industry practice, frontend projects, URLs, and general file naming tend to favor hyphens (kebab-case, such as `sim-gui`), while most of our current files and directories use underscores (snake_case, such as `sim_engine`).

A question came up during development: should the existing underscore naming be unified to the hyphenated style widely recommended in frontend and URL conventions?

---

## Decision

**Keep the status quo and reject a project-wide rename. Across the entire codebase, underscores (snake_case) are the preferred convention for directory and file naming.**

### Reasoning

1. A hard constraint from Python module imports: this project includes core Python logic (`sim_engine`). Python's syntax requires that a directory or file name imported as a module (for example `import sim_engine`) contain only letters, digits, and underscores. Using a hyphen (`-`) causes a Python syntax error, since it would be parsed as a minus sign.
2. Keeping the project internally consistent: to keep the frontend, backend, and documentation style unified, since the Python side must use underscores, having every other part (including frontend directories such as `sim_gui` and `game`, and the Markdown documents under `docs/`) also use underscores avoids a split style within the same code repository. Consistency within the project outweighs an external convention recommended by one particular language ecosystem.
3. An imbalance between the cost and benefit of renaming: the system currently runs well overall, and forcing a project-wide switch to hyphens would require restructuring a large number of code import paths, break the development server run from the terminal, break configuration files that depend on relative paths, and produce a large number of meaningless rename entries in the git history, a poor trade.

---

## Follow-On Convention and Impact

* New file/directory naming: going forward within this project (frontend and backend code, and Markdown documentation alike), whenever a word separator is needed, keep using an underscore `_`.
* Special exemption: file names mandated by a specific technology ecosystem's own standard are exempt (such as Node.js's `package-lock.json`, or continuous-integration configuration files), and such files keep the naming their own ecosystem requires.
