# 0121 - Cross-step_unit Parameter Conversion: Linear Division Versus a Root, Distinguished by Dynamics-Term Type

**Date**: 2026-06-25
**Status**: Adopted
**Related**: [0104](0104-2026-06-16_model_step-unit-per-formula-and-sim-step-size.md) (per-formula step_unit), [0105](0105-2026-06-16_model_step-unit-conditional-and-deprecate-dt.md)

---

## Background

ADR 0104 established the conversion `step = simulation.step_size / formula.step_unit`, but this mechanism only converts the formula's own `step` symbol; it does not convert the literal parameter values embedded in a dynamics expression. If a modeler derives a parameter value at a daily rate but uses it in a formula with `step_unit: hour`, the engine does not automatically detect or correct this mismatch; the conversion responsibility rests entirely with the modeler.

This kind of bug, a day-scale parameter plugged directly into an hour-scale formula, firing at full strength every single hour, a 24x amplification, has already been found independently in several models:

- `papers/s1/ckd_protein/ckd_protein_sim.yaml`: `beta0`/`mu_d`/`mu_u` (found 2026-06-20; three opt scenarios were consequently at 0% feasible throughout)
- Additional similar cases found during an internal model-library review, where a parameter derived as a day-granularity rate was plugged directly into an hour-granularity formula, driving the state variable to its upper bound or locking it prematurely.

Several cases had previously been masked by rounds of "manually tuning the coefficient smaller to force a plausible-looking curve," which drifted further from the original literature value with each attempt and never addressed the root cause. Fixing this surfaced a more subtle question: does a simple "divide by 24" apply exactly to every such parameter? The answer is no; it depends on which category of dynamics term the parameter sits in.

## Decision

### 1. Split into two categories by dynamics-term type

Category A, a state-independent flux term: the coefficient does not itself depend on the same state variable being updated (it does not form a feedback structure of "the state decaying/recovering toward a target"), of the form:

```
X: X + rate * f(other variables) * step
```

Example: the term `+ gamma_IR_liverfat * max(0, HOMA_IR - 2.5) * step` in `liver_fat_dynamics` (this term driving `liver_fat` does not itself depend on `liver_fat`).

Cross-step_unit conversion here is exactly linear: a daily rate divided by 24 gives the hourly rate. This flux stays approximately constant within any given hour (depending on how fast the other state variables change on an hourly scale, usually far slower than a 24-hour cycle), so the sum across 24 hourly steps equals exactly the result of 1 daily step.

Category B, a self-exponential decay or recovery term: the coefficient multiplies the difference between the state itself and some target (which can be 0), of the form:

```
X: X - k * (X - target) * step
```

Example: the term `- beta0 * GFR * step` in `ckd_protein_sim.yaml`'s `gfr_decline` (a special case with target=0); another similar term at roughly $k_{day}=0.15$ was found during the internal model-library review. This is the forward-Euler discretization of the first-order linear ODE $dX/dt=-k(X-\text{target})$. Converting $k_{day}$ to $k_{hour}$ is not a linear division; the compound effect of 24 hourly steps is multiplicative ($(1-k_{hour})^{24}$), and for it to equal the effect of 1 daily step ($1-k_{day}$), it must satisfy:

$$k_{hour} = 1-(1-k_{day})^{1/24}$$

(More generally, converting from a coarse granularity $n_{large}$ to a fine one $n_{small}$, with $n=n_{large}/n_{small}$: $k_{small}=1-(1-k_{large})^{1/n}$.)

The linear approximation $k_{hour}\approx k_{day}/24$ has negligible error only when $k_{day}$ is small; at $k_{day}=0.15$, the two differ by about 8%, which is not negligible.

### 2. The two computations cost the same; category B must use the exact formula

Both a root operation and a division are a single floating-point instruction computed once when the model loads, with no performance difference, so there is no reason to choose the linear approximation for convenience (this is not a precision-versus-performance trade-off; it is purely a matter of getting it right or wrong). Decision: category B must use the exact root formula and does not accept a linear approximation; category A uses linear division (or lets the formula's own `step_unit` conversion handle it), which is already an exact solution.

### 3. Simply changing `formula.step_unit` is not an acceptable fix

`step_unit` is a property of the entire formula, not of a single parameter. If a formula mixes a term already correctly calibrated to hour with a term calibrated to day but never converted (all three real cases this time were exactly this kind of mix), changing the overall `step_unit` would incorrectly dilute the former by 24x as well. The correct fix is to change only that parameter's own `value` (updating the `unit` field and the conversion note in `description` together), keeping the formula's `step_unit` unchanged.

### 4. How to judge it (a practical procedure)

1. Check whether a parameter's `unit` field is annotated with a time unit different from the formula's own `step_unit` (such as `unit: 1/day` appearing in a formula with `step_unit: hour`).
2. If so, look at the form of the dynamics term the parameter sits in: is it "a coefficient times (a state variable minus a target)" (category B, needing a root) or "a coefficient times a function of other variables" (category A, linear division suffices)?
3. Convert to the new value per the matching category, write it back into the parameter's `value`, update `unit` to the formula's actual time granularity, and record the conversion basis in `description` (the original literature's daily rate, the conversion formula, and the result) for traceability; append a `severity: resolved` entry to `metadata.todo` describing the discovery and fix.

## Trade-Offs

Given up: trying to have the engine auto-detect or auto-convert parameter-level units, which would require the engine to understand each dynamics expression's physical structure (whether it forms a self-exponential decay), beyond LM format's design principle that "a formula is a mathematical expression, and the engine does not perform semantic inference" (the same spirit as ADR 0070's asteval sandbox: the engine does not interpret an expression's meaning, only executes it).

Gained: writing this class of bug's judgment method and fix formula into an explicit rule (see the "Formula Step-Size Rules" section of `docs/model.md`), so future new models do not repeat the same mistake; when a modeler uses a day- or year-granularity literature statement such as "a half-life of X days" or "declines by Y per year" in `description`, there is now a clear checklist: first confirm whether this dynamics term is category A or category B, then decide the conversion method.

## Model Files Known to Have Been Fixed Under This Rule

| File | Parameter | Category | Note |
|---|---|---|---|
| `s1/ckd_protein/ckd_protein_sim.yaml` | `beta0`, `mu_d`, `mu_u` | B (all of the form `-k*state*step`, a self-decay/loss term with target=0) | Fixed with linear division on 2026-06-20; $k_{day}$ is very small (< 0.002), so the linear approximation differs from the exact solution by < 0.1%, negligible, and was not recomputed exactly |
| 3 more similar parameters elsewhere in the internal model library | — | B x1, A x2 | Each converted and fixed per its matching category on 2026-06-25; see the internal records for detail |

## Related

- ADR 0104 - the per-formula `step_unit` mechanism itself
- The "Formula Step-Size Rules" section of `docs/model.md` - where this rule's modeling guidance lives
- The internal task record `2026-06-20_task_s2-minipaper-rewrite-eval.md` - the record of how this bug was found
- The internal task record `2026-06-25_task_s2-model-bugfix-handoff.md` - the fix execution checklist
