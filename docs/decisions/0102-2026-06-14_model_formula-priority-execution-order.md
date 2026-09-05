# 0102 - Clarifying Formula priority Execution Order and Within-Step Update Visibility

**Date**: 2026-06-14
**Status**: Implemented (documentation clarified, plus one model's comment/priority mismatch fixed)
**Category**: Model specification / simulation-engine semantics clarification

---

## Background

Both `docs/model.md` and `docs/LM_format_1.0.md` previously stated that "a lower `priority` value executes first" (lower = first), but the actual engine implementation (`sim_engine/src/model_structure/simulation.py`'s `_build_formula_cache`/`step()`, and `sim_engine/src/simulator_engine.py`'s `solve_ode`) both use:

```python
sorted(formulas.items(), key=lambda x: x[1].priority, reverse=True)
```

That is, a larger number executes first, the opposite of what the documentation said.

This is not a merely theoretical discrepancy: the comment on `gastric_emptying_and_growth` (priority 10) in `models/papers/s1/infant_breastfeeding.yaml` states that it "executes after `stomach_intake_and_overflow` (priority 0), reading the stomach milk volume after overflow correction," but under the actual `reverse=True` ordering, the priority-10 formula executes before the priority-0 one, the opposite of the physiological order the author described (overflow correction first, then gastric emptying).

In addition, `LM_format_1.0.md` section 3.3's original description, "execute all `formula:` blocks first, then all `dynamics:` blocks," also did not match the implementation: the engine makes a single pass over the sorted formulas, evaluating each formula internally in the fixed order `condition -> dynamics -> formula(dict) -> formula(string)`, not two global passes grouped by field type.

Within `step()`, each formula's `dynamics`/`formula(dict)` write-back takes effect immediately (`var.value` and `asteval.symtable` are updated in sync), so within the same step a formula executing later reads the new value a formula executing earlier just wrote, a Gauss-Seidel-style sequential update, not a snapshot of the previous step's state (Jacobi-style). This had never been documented before.

## Decision

Keep the code as it is (`reverse=True`, a larger number executes first), and update the documentation and affected models to match the code's actual behavior, without changing the engine code (several published models' `priority` values already implicitly assume this behavior, so changing the code has a larger blast radius with no independent benefit).

### 1. Documentation corrections

- `docs/model.md`:
  - The YAML schema comment changed to "a larger number executes first."
  - A new "Formula Execution Order (priority)" subsection added, explaining the global single sort, the fixed evaluation order within a single formula, and within-step sequential write visibility, with a feed_intake/gastric_emptying example.
- `docs/LM_format_1.0.md`:
  - Section 3.1's example comment `(lower = first)` changed to `(higher = first)`.
  - Section 3.3 Execution Order rewritten: a single pass, the `condition -> dynamics -> formula(dict) -> formula(string)` evaluation order, Gauss-Seidel sequential write semantics, and `bounds` clipped per variable at each write (not clipped once at the end of the step).

### 2. Fixing the affected model

`models/papers/s1/infant_breastfeeding.yaml`: swapped the `priority` values of `stomach_intake_and_overflow` (0 to 10) and `gastric_emptying_and_growth` (10 to 0), so the actual execution order matches the physiological order the author described (overflow correction first, then gastric emptying, reading the corrected stomach milk volume); the priority numbers mentioned in comments were updated to match.

`maternal_sleep_tracking` (20) and `lm_score_accumulation` (30)'s execution order relative to these two (still after both) is unaffected by this swap.

## Impact and Validation

- No `sim_engine` code was changed, so existing tests and simulation results are unaffected.
- The execution order of the two formulas in `infant_breastfeeding.yaml` has changed (before the fix, gastric_emptying executed before stomach_intake_and_overflow; after, the order is reversed), so the numeric result may change accordingly, which is this ADR's intended correction (making the simulation match the author's described physiological causal order); the next time this model runs `--sim`/`--opt`, its `baby_stomach_volume`/`baby_weight`/`spit_up_volume` trajectories should be checked for continued plausibility (no `_HOLD`/`metadata.todo` marker has been found on this model so far).
- A repository-wide `grep -rn "priority.*(executes first|executes after|executes before|before|after)"` matched only this one location; other models' `priority` comments do not depend on an execution-order description and need no adjustment.

## Related

- `docs/model.md` - the new "Formula Execution Order (priority)" subsection, after the formula step-size rules section
- `docs/LM_format_1.0.md` sections 3.1, 3.3
- `models/papers/s1/infant_breastfeeding.yaml`
