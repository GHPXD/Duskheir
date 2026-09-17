---
name: game-balance
description: "Numerically balances engine-agnostic weapons, units, encounters, progression, loot, difficulty, and economies using measurable targets, spreadsheets, curves, probability, simulation, telemetry, and playtests. Use for tuning values and distributions; defer mechanic ideation to game-mechanics-design and cross-system structure to game-systems-design."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Game Balance

Balance means fitting numbers and rules to an intended experience for defined players, contexts, and time horizons. It does not mean making every option equal.

Remain engine-agnostic. Put tunable inputs, formulas, assumptions, scenarios, and checks in inspectable artifacts. Write implementation guidance only after the model and evidence are clear.

## Core Principles

- Begin with a measurable design target, not a preferred number.
- State the population, skill band, context, and horizon for every balance claim.
- Preserve meaningful differences. Options can have equal expected value and still differ in risk, timing, information, execution burden, or situational fit.
- Treat player choice as evidence only when the choice was available, understood, and credible.
- Keep raw inputs, derived metrics, player-facing display values, and observed outcomes separate.
- Centralize shared parameters. Do not type the same balancing constant into many formulas.
- Use a reference fulcrum to compare a large family of objects, then cross-test the unusual interactions.
- Choose curve shape from the desired marginal change, not from habit.
- Analyze an economy as flows and stocks across time, not as a price list.
- Analyze random outcomes by distribution and tail risk, not expected value alone.
- Use deterministic checks first, simulation for interacting or stochastic behavior, playtests for experience, and telemetry for behavior at scale.
- Change one causal lever or coherent lever bundle at a time and retain a rollback path.

## Intake

Inspect existing rules, spreadsheets, data exports, code/configuration, test results, simulations, and telemetry definitions before changing values. Establish:

- Balance question and observed symptom.
- Intended player experience and definition of success.
- Player segments, skill levels, modes, maps, encounters, and progression stage.
- Relevant time horizon: action, encounter, session, day, season, or lifetime.
- Candidate choices and when each is available.
- Attributes, formulas, dependencies, units, ranges, rounding, and caps.
- Current source of truth and data pipeline.
- Available evidence and its sample/context limitations.
- Production, fairness, accessibility, business, and legal constraints.

If information is missing, label assumptions and design the smallest measurement or experiment that can replace them.

## Workflow

### 1. Write a Balance Contract

Use a testable statement:

```text
For [player segment] in [context] over [horizon],
[primary metric] should remain within [target range],
while [guardrail metrics] remain within [ranges].
We will change [decision] if [evidence threshold] is crossed.
```

Examples of valid target dimensions include time to resolution, success rate by skill band, choice diversity by context, recovery opportunity, progression time, resource surplus, reward drought, and perceived clarity.

Do not set equal pick rate or 50% win rate by default. Explain why a target fits the game's purpose.

### 2. Define the Comparison Context

List the conditions under which values are comparable:

- Player skill and knowledge.
- Opponent or challenge profile.
- Map, range, team size, and mode.
- Starting resources and persistent power.
- Input method, latency, and execution assumptions.
- Duration and stop condition.

A global average across incompatible contexts is not a balance result. Segment first, aggregate second.

### 3. Build the Attribute Model

For each object and attribute, define meaning, unit, valid range, source, visibility, mutability, and mechanics that consume it. Mark attributes as:

- Input: directly tuned.
- Derived: calculated from inputs.
- Context: supplied by a scenario.
- Outcome: measured from play or simulation.
- Display: rounded or transformed for players.

Use [Quantitative Modeling](references/quantitative-modeling.md) for the balance brief, workbook layout, attribute worksheet, normalization, fulcrums, formulas, curves, and original worked example.

### 4. Structure the Spreadsheet or Model

At minimum, separate:

1. Read-me, assumptions, units, and change history.
2. Parameters and named constants.
3. Entity inputs.
4. Curves and lookup tables.
5. Derived calculations.
6. Representative scenarios.
7. Invariants and error checks.
8. Results, distributions, and charts.
9. Raw simulation or playtest imports.

Use one row per comparable object and one column per attribute. Validate input types and bounds. Keep intermediate calculations visible while debugging; hide or group them only after verification.

### 5. Establish Ranges and a Fulcrum

Normalize unlike units for comparison:

```text
normalized = clamp((value - minimum) / (maximum - minimum), 0, 1)
value = minimum + normalized * (maximum - minimum)
```

Choose a representative, intentionally ordinary object or scenario as the fulcrum. Validate it in common contexts before deriving a family around it.

Compare each variant with the fulcrum using outcome metrics, not raw attribute sums. Equal weighted scores are hypotheses, not proof. Cross-test extremes, counters, combinations, and options whose rules differ from the fulcrum.

### 6. Select Curves by Marginal Effect

Inspect both total values and change per step.

- Linear: constant absolute increment.
- Exponential: constant proportional increment; use for compounding or rapidly rising costs.
- Power curve: adjustable acceleration or deceleration.
- Saturating or logarithmic: large early gains followed by diminishing returns.
- Piecewise: distinct phases with explicit transition points.

Check the first, middle, last, and beyond-cap values. Plot the curve, its step-to-step difference, and time-to-earn or time-to-defeat. Keep internal precision separate from display rounding.

### 7. Model Intertwined Attributes

Reduce dependent attributes to outcome measures before comparison. Common examples:

```text
throughput = amount_per_action * actions_per_second * success_probability * uptime
time_to_goal = required_amount / net_gain_rate
expected_damage = hit_probability * damage_on_hit
effective_durability = health / (1 - damage_reduction)
```

State assumptions about reloads, downtime, overkill, target resistance, movement, resource limits, and player execution. A single composite score is useful for screening but can conceal burst, variance, reach, control, and team utility.

### 8. Model Economies and Randomness

Use [Economies and Probability](references/economies-and-probability.md) to map sources, sinks, stocks, conversions, affordability, weighted outcomes, expected value, variance, droughts, and dependent events.

For economies, calculate net flow per relevant player-time unit and by progression cohort. Check inflation, stagnation, hoarding, mandatory upkeep, and wealth concentration.

For random systems, calculate outcome probabilities, expected payout, spread, percentiles, repeated-failure probability, and dependency. Model guarantees or escalating odds as stateful systems, not independent rolls.

### 9. Check Deterministic Extremes

Before simulation or human testing, verify:

- Minimum, maximum, zero, cap, and one-step-past-cap values.
- Empty and full stocks.
- Fastest and slowest legal rates.
- Simultaneous events and tie resolution.
- Rounding boundaries.
- Impossible or invalid input handling.
- Best-case combo, worst-case combo, and self-sustaining loops.
- Start-from-zero and recovery-from-deficit behavior.

Purposefully test absurd values in an isolated copy. Extreme tests reveal hidden dependencies and engine or rule limits quickly.

### 10. Simulate Interacting Outcomes

Use simulation when analytical formulas become unwieldy because of state, policies, dependencies, or repeated randomness. Follow [Simulation, Telemetry, and Tuning](references/simulation-telemetry-and-tuning.md).

Define agent policies and skill assumptions explicitly. Run multiple deterministic seeds, report distributions and confidence, and verify simple cases analytically. A simulated agent's preferred option proves only what is optimal for that policy.

Use sensitivity sweeps to identify which inputs move outcomes most and which interactions reverse conclusions.

### 11. Playtest the Decision

Test whether players can perceive the tradeoff and use it, not merely whether model outputs are close.

Observe:

- Choice time and comparison behavior.
- Choice share conditional on availability and context.
- Strategy switching as conditions change.
- Disagreement supported by coherent reasoning.
- Options never considered or abandoned after one use.
- Prediction accuracy and perceived fairness.
- Recovery behavior after bad outcomes.

Fresh players test legibility. Experienced players test depth and optimization. Separate confusion from thoughtful hesitation.

### 12. Instrument Telemetry Before Scaling

For each metric, define the decision it informs, event, properties, denominator, cohort, and retention period. Include build and balance-data versions so results can be attributed to the correct rules.

Record opportunity as well as action. Pick rate is meaningless if the option was not offered or affordable. Record relevant state before and after an economy transaction or encounter.

Collect only necessary data, use appropriate consent and privacy practices, and do not infer player intent from one aggregate metric.

### 13. Tune as an Experiment

For each tuning pass:

1. State the diagnosed cause.
2. Choose the lowest-impact lever that can affect it.
3. Predict primary and guardrail movement.
4. Change one lever or inseparable bundle.
5. Recalculate the model and invariants.
6. Re-run deterministic cases and fixed-seed simulations.
7. Playtest or release to a controlled cohort when warranted.
8. Compare against the prior version and segments.
9. Keep, revise, roll back, or gather more evidence.
10. Record the decision and update the source of truth.

## Diagnostic Signals

- **Dominant option:** check context-adjusted outcomes, information advantage, execution cost, availability, and synergy before nerfing its headline value.
- **Unused option:** verify discoverability and eligibility, then compare its situational payoff and cognitive cost.
- **Snowball:** measure how current advantage changes future earning, denial, information, or action capacity. Add saturation, exposure, upkeep, alternate value paths, or recovery opportunities.
- **Failure spiral:** find the point where loss removes the resources needed to act. Preserve a floor or provide a costly but credible recovery path.
- **Grind:** graph time-to-goal and meaningful decisions per interval. Reduce repetition, adjust earning/cost curves, or add strategically distinct routes.
- **Inflation:** compare source and sink rates by cohort, holdings percentiles, price movement, and purchase relevance.
- **Stagnation:** check whether scarcity or fear makes saving always preferable to spending.
- **RNG frustration:** inspect failure-streak and reward-drought percentiles, not only advertised chance or mean payout.
- **Model/play mismatch:** audit omitted constraints, player execution, feedback, context mix, and implementation rounding.
- **Aggregate contradiction:** segment by skill, progression, map, mode, party, build, and exposure.
- **Telemetry jump:** check schema, balance version, population mix, event opportunity, and instrumentation before attributing it to design.

## Failure Modes

- Tuning without a measurable contract.
- Equating balance with sameness or equal popularity.
- Adding raw attributes that never affect a decision or outcome.
- Comparing unlike units without normalization or context.
- Treating a weighted score as proof of equal performance.
- Choosing a curve by appearance while ignoring marginal change and time cost.
- Hard-coding repeated constants throughout a workbook.
- Using averages that hide tails, outliers, or mixed cohorts.
- Treating dependent or guaranteed random events as independent.
- Simulating an unrealistic policy and calling the result player behavior.
- Changing several unrelated values, making attribution impossible.
- Correcting a local symptom with a global multiplier.
- Collecting telemetry without a decision, denominator, or version.
- Trusting statistically precise data that measures the wrong behavior.
- Shipping hidden adaptive difficulty that violates the game's fairness promise.
- Rounding early and allowing cumulative drift between model and implementation.

## Required Deliverables

Unless the user requests a narrower artifact, produce:

- Balance contract, player segments, contexts, horizon, and guardrails.
- Attribute/data dictionary with units, ranges, ownership, and formulas.
- Inspectable model or spreadsheet layout with centralized parameters.
- Fulcrum and representative scenario set.
- Curve, economy, and probability analysis where applicable.
- Deterministic edge checks and invariant results.
- Simulation or playtest plan with assumptions and outputs.
- Telemetry question-to-event plan.
- Ranked diagnosis with causal evidence and uncertainty.
- Tuning proposal, expected movement, rollback condition, and change log entry.

## Verification

- Targets are measurable and scoped to players, context, and time.
- Inputs, derived values, display values, and observations are distinct.
- Units are compatible, formulas are labeled, and rounding order is explicit.
- Shared constants have one authoritative location.
- Normalization round-trips at minimum, midpoint, and maximum.
- The fulcrum works in ordinary scenarios; extremes and unique rules are cross-tested.
- Curves satisfy endpoints, monotonicity or intended reversals, caps, and marginal-change goals.
- Economy stocks reconcile with sources, sinks, transfers, and starting balances.
- Probability weights normalize; expected value and tail risks are checked.
- Deterministic edge cases pass before stochastic tests.
- Simulations are reproducible, policies are documented, and results converge sufficiently for the decision.
- Telemetry records opportunity, context, version, action, and outcome without unnecessary data.
- Model predictions are compared with playtest or live evidence.
- Guardrails and regressions are checked after each change.
- The source of truth, decision record, and rollback values are current.
