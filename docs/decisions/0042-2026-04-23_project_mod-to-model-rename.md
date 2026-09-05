# ADR 0042 - A Project-Wide Rename from mod to model
**Date**: 2026-04-23
**Status**: Implemented

---

## Background

The word `mod` carried a double meaning in the project:

- Game mod: a custom historical scenario made by a community user (the conventional abbreviation for game modification).
- An abbreviation of model: a dynamics model in the simulation framework (a YAML file, located under `mods/`).

This caused confusion in several places:
- `mod_design.md` / `mod_requirements.md` / `mod_impl.md` were designed to describe the dynamics models, but a reader (and an AI-assisted tool) would misread them as being about a "game MOD system."
- The directory `mods/models/` was semantically redundant ("a mod's models"), and `mods/` implied "MODs welcome," which does not match the non-game, research-modeling positioning.
- The Python modules `mod_structure`, `mod_generator`, `mod_merger` and others were likewise contaminated by this ambiguity.

Whether `sim_xxx` should also be changed to `simulation_xxx` was discussed at the same time, and the conclusion was not to change it (see below).

---

## Decision

### Decision 1: renaming docs files

| Old file | New file | Note |
|--------|--------|------|
| `mod_requirements.md` | `model_requirements.md` | The model data specification (the required `description`/`reference`/`comments` fields) |
| `mod_design.md` | `model_design.md` | The YAML modeling specification, split out from `sim_design.md` |
| `mod_impl.md` | `model_impl.md` | Model-related implementation notes |

The Game MOD community content that used to be in `mod_xxx.md` moved into `game_design.md` and `game_requirements.md`, and `mod_xxx.md` was then deleted.

The YAML format specification chapter in `sim_design.md` was extracted into the newly created `model_design.md`, and `sim_design.md` kept its original simulation-engine design content.

### Decision 2: renaming the directory structure

| Old path | New path |
|--------|--------|
| `mods/` | `models/` |
| `mods/models/` | `models/source/` |
| `mods/stories/` | `models/stories/` |
| `mods/scenarios/` | `models/scenarios/` |

Reason for `mods/models/` becoming `models/source/`: the inner `models` overlapped semantically with the outer `mods`; `components` more accurately describes "a library of reusable dynamics components."

### Decision 3: renaming code directories and modules

| Old name | New name |
|--------|--------|
| `plugins/preprocessors/mod_generator/` | `model_generator/` |
| `plugins/preprocessors/mod_merger/` | `model_merger/` |
| `sim_engine/src/mod_structure/` | `model_structure/` |

Every Python file's `mods_directory` parameter default value changed from `"mods"` to `"models"`; the import path `from .mod_structure import` changed to `from .model_structure import`.

### Decision 4: updating frontend and backend path strings accordingly

- `api_server.py`: `PROJECT_ROOT / "mods"` became `/ "models"`; the API route `/api/mods/` became `/api/models/`; the root node key returned by `/api/files` changed from `'mods'` to `'models'`.
- `game/vite.config.ts`: `modsDir` now points to `'models'`, and the `/mods/` URL handler became `/models/`.
- `game/src/App.tsx`, `StorySelect.tsx`: the storyPath prefix `` `mods/${p}` `` became `` `models/${p}` ``.
- Various `sim_gui` components: `n.key === 'mods'` became `'models'`; `n.key === 'models'` (the former inner components directory) became `'components'`; `ModsManager`'s visibility filter changed from `models/` + `scenarios/` to `components/` + `scenarios/`.

---

## Decision 5: keeping sim_xxx unchanged

The possibility of renaming `sim_xxx` (filenames, variable names, parameter names) to `simulation_xxx` was discussed.

Conclusion: keep the `sim` prefix, do not change it.

Reasoning:
- `sim` is a recognized abbreviation in the simulation-engineering field (used by MATLAB Simulink, SimPy, OpenSim, etc.), with no risk of ambiguity.
- `mod`'s ambiguity came from a genuine conflict with the meaning of game mod; `sim` has no corresponding source of ambiguity.
- The scope of change would be far larger than the `mod` rename (URL paths, Python class names, parameter names, frontend component names), a much wider surface for errors, with zero benefit.

---

## Result

```
models/                     <- formerly mods/
  components/               <- formerly mods/models/ (reusable dynamics components)
  scenarios/                <- simulation scenario configuration
  stories/                  <- game story packages

plugins/preprocessors/
  model_generator/          <- formerly mod_generator/
  model_merger/              <- formerly mod_merger/

sim_engine/src/
  model_structure/          <- formerly mod_structure/
```

`mod_xxx.md` has been removed from docs, its Game MOD content folded into `game_xxx.md`, and the YAML modeling specification split out as its own `model_design.md`.

No `mods/` path or `mod_` module prefix appears anywhere in the code or documentation any longer. The `sim_xxx` naming is unchanged.
