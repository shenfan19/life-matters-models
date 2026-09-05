# models/test_fixtures/invalid — Error-detection test cases

## Purpose

Every file under `test_fixtures/invalid/` is **deliberately broken**, specifically to verify the engine's error-detection mechanism: not only must it correctly load a structurally valid model (see `test_fixtures/valid/`), it must also **fail reliably** when it encounters a structural error, a circular import, a misconfigured evidence block, an invalid date, and the like, and expose the specific reason to the caller through a normal call path (`ReferenceEngine.load_models()` / `run_simulation()`, etc., not just a log line).

**Each file deliberately breaks exactly one thing**, keeping everything else structurally valid, so each file corresponds to one clear validation branch; when validation logic changes, you only need to check whether this one file's test still fails as expected (the same principle as "one folder per variable" in `test_verification/models/README.md`).

The corresponding pytest assertions are in `test_verification/errors/`, and each file notes in its `metadata.description.result` the expected error-message substring and the corresponding test file. **These models will never, and should never, be fixed to run successfully** — a `--sim`/`--opt` failure or a FAIL reported by `cli/batch.py` is by design, not a regression.

## File list

| File | Validation triggered | Validation location |
|------|-----------|---------|
| `test_invalid_step_size.yaml` | `simulation.step_size` must be a positive number | `validator.py` `Validator.validate_model` |
| `test_invalid_optimization_missing_method.yaml` | `optimization.method` is required | `validator.py` `Validator.validate_model` |
| `test_invalid_equation_undefined_var.yaml` | `dynamics` references an undeclared variable | `validator.py` `validate_equations` (extracts variables via AST) |
| `test_invalid_equation_deprecated_dt.yaml` | `dynamics` uses the deprecated symbol `dt`, should use `step` instead | `validator.py` `validate_equations` |
| `test_invalid_import_circular_a.yaml` + `_b.yaml` | Circular-import detection (a and b import each other) | `loader.py` `Loader._load_model_data` |
| `test_invalid_import_escapes_root.yaml` | A relative import climbs above the `models/` root | `loader.py` `Loader._load_model_data` |
| `test_invalid_evidence_name_collision.yaml` | An `evidence` name collides with `variables` | `loader.py` `Loader._apply_model_data` |
| `test_invalid_evidence_missing_baseline_ref.yaml` | An `rr`/`or` evidence entry using `applies_to` is missing `baseline_ref` | `loader.py` `Loader._apply_model_data` |
| `test_invalid_yaml_not_dict.yaml` | The top-level YAML must be a mapping, not a list/scalar | `loader.py` `Loader._load_model_data` |
| `test_invalid_date_range.yaml` | `end_date` must not be earlier than `start_date` | `validation.py` `validate_simulator_dates` (checked only at run time; loading/`validate_model()` does not check this, see the description inside the file) |

## Adding a new error-detection case

1. First confirm the target validation branch exists in the engine code (`validator.py` / `loader.py` / `validation.py`); do not write a fixture for a validation that doesn't exist.
2. Copy the structurally closest existing `test_invalid_*.yaml` as a template, changing only the minimal fields needed to trigger the target validation.
3. In `metadata.description`, clearly state: what was deliberately broken, what substring the expected error message contains, and which `test_verification/errors/*.py` file it corresponds to.
4. Add the assertion in the corresponding file under `test_verification/errors/`, and add a row to the table above.
