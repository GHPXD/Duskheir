# Quantitative Modeling

Use this reference to create measurable targets, maintainable spreadsheets, comparable attributes, stable fulcrums, and intentional progression curves.

## Balance Brief

```text
Design decision:
Player segment:
Context and available choices:
Time horizon:
Primary experience target:
Primary metric and target interval:
Guardrail metrics and intervals:
Known asymmetries that should remain:
Current baseline and evidence quality:
Model assumptions:
Decision threshold:
Owner and review date:
```

Prefer intervals over exact points unless the rule requires exact conservation. A target of 45-60 seconds to resolve an ordinary encounter is more honest and useful than 52.3 seconds when player behavior drives variation.

## Attribute Dictionary

| Field | Purpose |
| --- | --- |
| Attribute name | Stable, unambiguous identifier |
| Design meaning | What concept it abstracts |
| Mechanics usage | Which rules consume it |
| Unit | Seconds, meters, points, percentage, items, and so on |
| Valid range | Technical and design bounds |
| Default/fulcrum | Ordinary reference value |
| Kind | input, derived, context, outcome, or display |
| Visibility | hidden, approximate, or exact to players |
| Persistence | action, encounter, session, or account |
| Formula/source | Authoritative origin |
| Owner | Who may change it |

Remove an attribute when no rule, decision, display, or analysis uses it. Combine attributes whose differences never produce different player choices.

## Workbook Layout

Use a structure that another designer can audit without oral explanation:

```text
00_ReadMe       purpose, owners, units, conventions, versions, change log
01_Parameters   named constants, range endpoints, weights, caps
02_Entities     one row per comparable object, input attributes only
03_Curves       progression tables and marginal changes
04_Derived      intermediate and final formulas
05_Scenarios    controlled contexts and assumptions
06_Checks       invariants, bounds, missing values, duplicate IDs
07_Results      summaries, quantiles, charts, comparisons
08_RawRuns      imported simulation, playtest, or telemetry extracts
99_Scratch      disposable calculations that are not authoritative
```

### Conventions

- Use named parameters or stable references instead of repeated literals.
- Label every table, unit, formula output, and data version.
- Keep one row per object and one column per attribute inside data tables.
- Do not use blank rows as category markers; add a category column.
- Validate types, enumerations, and ranges at input.
- Protect formula cells and make tunable cells visually distinct with a documented convention.
- Break long formulas into named intermediate columns.
- Separate raw source data from cleaned data and calculated output.
- Add invariant checks whose correct result is already known.
- Keep full internal precision; round only in dedicated display columns or at a specified implementation boundary.

## Useful Checks

```text
IDs are unique
required cells are nonblank
minimum <= value <= maximum
probabilities sum to 1 within tolerance
source total - sink total = stock change
normalized values remain in [0, 1]
curve is monotonic where intended
display rounding does not change tier/order unexpectedly
lookup keys all resolve
simulation and model versions match
```

Use a small numeric tolerance for floating-point checks rather than exact equality where appropriate.

## Normalized Ranges

Map a native unit into a common design range:

```text
n = clamp((x - min_x) / (max_x - min_x), 0, 1)
x = min_x + n * (max_x - min_x)
```

If `max_x = min_x`, the range is undefined. Fix the model rather than dividing by zero.

Normalization helps compare unlike scales, but it does not make unlike attributes equally valuable. Weight or combine them only after measuring how they affect outcomes.

### Range Worksheet

```text
Attribute:
Native unit:
Technical minimum/maximum:
Playable minimum/maximum:
Ordinary operating band:
Player-visible scale:
Reason for endpoints:
Behavior below/above endpoints:
Rounding/display rule:
Evidence used to revise range:
```

Reserve room beyond ordinary values when temporary effects, future content, or injuries need it. Test whether players can perceive the smallest meaningful step.

## Reference Fulcrum

A fulcrum is an ordinary, stable comparison object or scenario. It need not ship. It should represent the middle of intended play, not the arithmetic middle of every raw range.

### Fulcrum Worksheet

```text
Object family:
Ordinary use context:
Outcome targets:
Input attributes:
Scenario assumptions:
Known strengths intentionally absent:
Known weaknesses intentionally absent:
Tests required before freezing:
Version and evidence:
Conditions that permit revision:
```

Workflow:

1. Start with middle-of-range inputs.
2. Test the fulcrum against itself or against the ordinary scenario.
3. Adjust one input at a time until outcome targets and feel are credible.
4. Test across representative contexts.
5. Freeze a version and derive variants from it.
6. Compare each variant to the fulcrum with the same scenarios.
7. Cross-test extremes, counters, synergies, and unique mechanics.

Do not permanently freeze a broken reference. Reopening it should be rare, explicit, versioned, and followed by regression of every derived family.

## Weighted Screening Score

Normalize first, then weight:

```text
screening_score = sum(weight_i * normalized_attribute_i)
```

Document whether higher is always better. Invert costs or drawbacks when needed:

```text
normalized_benefit_of_low_cost = 1 - normalized_cost
```

Use the score to identify likely outliers and seed tests. Do not use it as final proof because interactions, thresholds, timing, and context violate simple additivity.

## Intertwined Attributes

Derive a gameplay outcome before assigning value.

### Throughput

```text
actions_per_second = 1 / cycle_seconds
throughput = amount_per_action * actions_per_second * success_probability * uptime
```

Include reload, recovery, travel, resource starvation, or animation lock in `cycle_seconds` or `uptime` when they matter.

### Time to Resolution

For deterministic discrete hits:

```text
hits_required = ceil(target_health / damage_per_hit)
time_to_defeat = windup + (hits_required - 1) * cycle_seconds
```

The final hit has no following cooldown, so multiplying every hit by the full cycle can overstate time.

For expected continuous output:

```text
estimated_time_to_defeat = target_health / effective_damage_per_second
```

Use simulation when misses, critical hits, reloads, overkill, control effects, or target behavior make the distribution important.

### Effective Durability

For damage reduction `r` in `[0, 1)`:

```text
effective_durability = health / (1 - r)
```

This estimate fails when reduction is type-specific, capped per hit, bypassed, or changed during the encounter. Model those scenarios separately.

### Progression Time

```text
steps_to_goal = required_resource / expected_resource_per_step
time_to_goal = required_resource / expected_resource_per_minute
cumulative_time(rank) = sum(time_to_complete_each_prior_rank)
```

Graph per-rank time and cumulative time. A gentle-looking cost curve can produce an exhausting cumulative curve.

## Curve Recipes

Let `x` start at zero unless stated otherwise.

### Linear

```text
y(x) = start + step * x
```

Use when each step should add the same absolute amount. Solve a line through endpoints:

```text
step = (end - start) / number_of_steps
```

### Exponential

```text
y(x) = start * ratio^x
ratio = (end / start)^(1 / number_of_steps)
```

Use when each step should add the same percentage. `start` and `end` must be positive for this form.

### Power

```text
y(x) = start + scale * x^p
```

- `p > 1`: accelerating absolute gains.
- `0 < p < 1`: diminishing absolute gains.

### Saturating Return

```text
y(x) = cap * x / (half_point + x)
```

At `x = half_point`, output reaches half of `cap`. Use for resistance, reputation benefit, accuracy support, or any value that should approach but not reach a ceiling.

### Logarithmic Return

```text
y(x) = start + scale * ln(1 + x)
```

Use for strong early benefit with a long, slowly improving tail. Keep `x >= 0`.

### Piecewise

```text
y(x) = early_rule(x)   when x < breakpoint
y(x) = late_rule(x)    when x >= breakpoint
```

Use when the design intentionally changes phase. Check continuity at the breakpoint unless a visible jump is desired.

## Curve Review Worksheet

```text
Design purpose:
Input and output units:
Domain:
Required start/middle/end values:
Desired marginal change:
Chosen family and why:
Cap/floor:
Breakpoints:
Display rounding:
Time-to-earn assumptions:
First five outputs:
Middle outputs:
Last five outputs:
One step beyond domain:
Largest absolute step:
Largest percentage step:
```

Plot:

- Total output `y(x)`.
- Absolute marginal change `y(x) - y(x-1)`.
- Proportional change `y(x) / y(x-1) - 1`.
- Player time or actions required to obtain each step.

## Worked Example: Salvage Tools

Three tools in a salvage game should produce similar long-run material throughput in their favored ordinary context while supporting different risk preferences.

```text
throughput_per_minute = payload * success_probability / cycle_seconds * 60
```

| Tool | Payload | Success | Cycle seconds | Expected per minute |
| --- | ---: | ---: | ---: | ---: |
| Clamp | 5 | 0.90 | 15.0 | 18.0 |
| Needle | 3 | 0.75 | 7.5 | 18.0 |
| Hauler | 9 | 0.60 | 18.0 | 18.0 |

`Clamp` is the fulcrum. Equal means do not make the tools interchangeable:

- Needle gets more attempts and recovers quickly from a miss, but requires more actions.
- Hauler has larger bursts and longer droughts, making interruption more costly.
- Clamp offers a steadier middle case.

Required follow-up models:

- Energy consumed per successful material.
- Lost payload when interrupted mid-cycle.
- Storage overfill and overkill waste.
- Player success rates rather than assumed success rates.
- Short encounter percentiles, where the means may not converge.
- Upgrade interactions that alter cycle, success, or payload together.

This example demonstrates the role of a fulcrum and an outcome metric. It does not prove balance until the contextual guardrails and player experience are tested.

## Verification

- All formulas use compatible units.
- Range mappings return 0, 0.5, and 1 at minimum, midpoint, and maximum.
- Inputs and derived values are not mixed in editable columns.
- Weighted scores use documented normalized direction and weights.
- The fulcrum is versioned and tied to scenario outcomes.
- Curves hit required endpoints and have intentional marginal behavior.
- Rounding is applied only at the stated boundary.
- Every chart includes labels, units, and the relevant domain.
- Model results are compared with at least one independent calculation or controlled test.
