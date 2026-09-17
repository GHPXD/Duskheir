# Simulation, Telemetry, and Tuning

Use simulation to explore repeated interacting rules, telemetry to observe real behavior at scale, and controlled tuning to connect evidence to reversible changes.

## Select the Evidence Method

| Method | Best for | Cannot establish alone |
| --- | --- | --- |
| Formula/enumeration | exact simple relationships and invariants | player strategy or perception |
| Deterministic trace | event order, boundaries, and edge cases | distribution across randomness |
| Stochastic simulation | distributions, repeated interactions, sensitivity | whether humans understand or enjoy it |
| Bot/agent tournament | policy matchups and exploit search | real player optimality |
| Moderated playtest | comprehension, decision process, and experience | population prevalence |
| Telemetry | behavior and outcomes at scale | causality or private intent by itself |
| Controlled live experiment | comparative causal evidence under stated assumptions | long-term effects outside its window |

Triangulate when the decision is costly or risky.

## Simulation Specification

```text
Decision to inform:
Model version:
Population/unit simulated:
Initial state distribution:
Time step and horizon:
Transition rules:
Random variables and distributions:
Agent policies and skill/noise assumptions:
Configurations compared:
Seeds and run count:
Stop conditions:
Metrics and quantiles:
Invariants:
Analytical cases used for verification:
Known omissions:
Decision threshold:
```

If agent behavior drives the result, treat policy assumptions as first-class inputs. Compare several policies rather than labeling one "the player."

## Engine-Agnostic Simulation Skeleton

```text
for each configuration:
    for each seed:
        random = seeded_generator(seed)
        state = initialize(configuration, random)

        while not finished(state):
            observations = expose_information(state)
            actions = choose_actions(observations, policies, random)
            events = resolve_actions(state, actions, random)
            state = apply_events(state, events)
            assert_invariants(state)
            record_step(configuration, seed, state, actions, events)

        record_run_summary(configuration, seed, state)

aggregate_distributions()
compare_configurations()
```

Keep transition logic separate from policy logic. This lets the same game rules run under random, greedy, scripted, adversarial, and noisy skill-tier policies.

## Policy Set

Useful baseline policies include:

- Random legal choice: catches gross dominance and dead actions.
- Greedy immediate reward: exposes short-horizon exploits.
- Greedy long-term value: tests investment and snowball behavior.
- Rule-of-thumb novice: uses visible signals with execution noise.
- Experienced heuristic: uses known synergies and context.
- Adversarial search: seeks stalls, loops, denial, and degenerate states.
- Scripted edge policy: forces rare transitions and boundary states.

Document information available to each policy. A policy must not read hidden state unavailable to the intended player.

## Reproducibility and Verification

- Seed every random stream and store the seed per run.
- Store model, parameter, and policy versions with results.
- Verify a no-randomness case against a hand calculation.
- Verify a small random case by complete enumeration when feasible.
- Assert conservation, bounds, and legal-transition invariants every step.
- Run the same seed before and after a change for paired debugging.
- Confirm results do not depend accidentally on iteration order.
- Inspect raw traces for selected best, median, worst, and anomalous runs.

## Run Count and Uncertainty

Choose runs based on stability of the decision metric, not a ritual number.

For an approximately independent sample mean:

```text
standard_error = sample_standard_deviation / sqrt(n)
approx_95_percent_interval = mean +/- 1.96 * standard_error
```

For an observed proportion `p_hat`:

```text
standard_error ~= sqrt(p_hat * (1 - p_hat) / n)
```

Use a more appropriate interval for small samples or extreme proportions. Report at least count, mean or rate, median, and relevant low/high quantiles. Heavy-tailed results often need many more runs than stable central outcomes.

Convergence procedure:

1. Run a small batch.
2. Double the cumulative run count.
3. Compare key estimates and quantiles with the prior batch.
4. Continue until the decision remains unchanged within a stated tolerance or budget.
5. Report unresolved uncertainty rather than hiding it.

## Sensitivity Analysis

Use sensitivity to find leverage and brittle conclusions.

### One-Factor Sweep

Vary one input across and slightly beyond its legal range while holding others fixed. Plot outcomes and marginal change. Good for monotonicity, thresholds, and caps.

### Grid Sweep

Vary two high-impact inputs together. Use a heat map to reveal interactions and regions where the preferred option changes.

### Scenario Sweep

Run all candidate options across named contexts: close/far, rich/poor, solo/team, novice/expert, early/late progression, and low/high latency where relevant.

### Random Parameter Sampling

Sample several uncertain parameters from plausible ranges. Use this to see whether a conclusion survives uncertainty, not to replace thoughtful scenario design.

Rank inputs by how much they move the decision metric, but inspect interactions before declaring one lever causal.

## Simulation Output Template

```text
Question:
Compared configurations:
Policies and assumptions:
Runs/seeds:
Primary result with uncertainty:
Median and tail results:
Invariant failures:
Sensitivity reversals:
Representative traces:
Mismatch with analytical expectations:
Known model gaps:
Decision supported:
Next human test:
```

## Telemetry Question Map

Never begin with "track everything." Begin with a decision.

| Decision | Metric | Denominator/context | Event | Required properties | Threshold/action |
| --- | --- | --- | --- | --- | --- |

Example:

```text
Decision: revise the Hauler tool if it is understood but nonviable
Metric: selection rate and successful material per minute
Denominator: sessions where Hauler was unlocked, shown, and affordable
Context: route type, player progression, prior uses, and build version
Events: option_presented, option_selected, salvage_cycle_resolved
Action: revise if use remains low and contextual outcome is below guardrail
```

## Event Envelope

Use consistent shared fields:

```text
event_name
event_schema_version
timestamp
build_version
balance_data_version
anonymous_player_id
session_id
mode/map/encounter
progression_band
skill_or_match_band, if valid
experiment_id and variant, if any
```

Decision events should usually include:

```text
decision_id
options_available
eligibility and affordability
choice
relevant_state_before
relevant_state_after
outcome
```

Economy events should include resource, amount, stock before/after, source/sink category, transaction context, and counterparty type. Do not record unnecessary personal or sensitive information. Establish consent, access, retention, and deletion practices appropriate to the project and jurisdiction.

## Metric Definitions

### Conditional Choice Share

```text
choice_share(option) = selections(option) / valid_presentations(option)
```

Do not divide by all sessions when the option was locked, hidden, unaffordable, or incompatible in many sessions.

### Contextual Outcome

```text
outcome_rate = successful_outcomes / valid_attempts
```

Stratify by relevant context and skill. Comparing raw win rate between an expert-only unlock and a starter option creates selection bias.

### Economy Flow

```text
net_flow = sum(source_amounts) - sum(sink_amounts)
```

Report per player-hour or another meaningful exposure unit, plus holdings and spending quantiles.

### Progression and Friction

- Time and attempts to milestone by cohort.
- Failure and abandonment at each opportunity.
- Repeated failure before success.
- Help, skip, or difficulty-change use.
- Resource shortfall when a milestone is presented.

Define starts, completions, timeouts, and idle-time handling precisely.

## Telemetry Validation

Before trusting live results:

1. Generate every event in a controlled test.
2. Verify required properties and units.
3. Reconcile event counts with an independent local counter.
4. Test retries, duplication, offline queues, reconnects, and clock issues.
5. Confirm build and balance versions propagate correctly.
6. Check that option-presentation denominators exist.
7. Run impossible-value and missing-property monitors.
8. Compare a small set of session replays or logs with reconstructed metrics.

Instrumentation bugs can look exactly like balance changes.

## Analysis Guardrails

- Show sample size and missing-data rate.
- Use medians and quantiles alongside averages.
- Segment before drawing a global conclusion.
- Distinguish correlation, prediction, and causal comparison.
- Account for exposure and survivorship; late-game data represents players who reached late game.
- Mark balance patches, content launches, promotions, and population changes.
- Predefine primary metrics for controlled experiments.
- Avoid repeated peeking and many uncorrected comparisons.
- Check practical effect size, not only statistical significance.
- Pair telemetry with qualitative evidence when interpretation or fairness matters.

## Tuning Proposal

```text
Observed symptom:
Affected segment/context:
Evidence and limitations:
Diagnosed causal mechanism:
Current parameter(s):
Proposed parameter(s):
Why this is the lowest-impact lever:
Expected primary movement:
Guardrails expected not to move:
Model/check results:
Fixed-seed simulation results:
Playtest/live experiment plan:
Exposure and observation window:
Keep/revise/rollback thresholds:
Dependencies and regression risks:
Owner:
```

## Tuning Log

| Version/date | Hypothesis | Change | Primary result | Guardrails | Decision | Evidence link |
| --- | --- | --- | --- | --- | --- | --- |

Retain old values. A rollback should be a data/version change, not an attempt to reconstruct forgotten numbers.

## Causal Tuning Sequence

When an option appears too strong or weak, investigate in this order:

1. Was it actually available and understood?
2. Which contexts and players selected it?
3. Is the measured outcome the intended value it provides?
4. Does selection itself correlate with skill, progression, or team composition?
5. Which rule creates the advantage: base output, uptime, information, safety, denial, synergy, or economy access?
6. Can a contextual counter or clearer cost preserve identity better than a flat numeric change?
7. What downstream curves, economies, tutorials, AI, content, and achievements depend on the lever?

## Failure Modes

- **Simulation matches itself:** transition and verification use the same faulty formula. Add an independent hand or enumerated check.
- **One policy declared optimal:** policy assumptions determine the winner. Compare policies and information constraints.
- **More runs hide model error:** confidence narrows around a biased model. Validate structure before scale.
- **Only means reported:** droughts, blowouts, and long tails disappear. Report distributions and representative traces.
- **Telemetry lacks opportunity:** popularity denominator is wrong. Instrument presentation, eligibility, and affordability.
- **Version mixing:** incompatible balance states are aggregated. Require build and balance-data versions.
- **Metric becomes target:** tuning improves a proxy while harming the experience. Keep player-facing guardrails and qualitative checks.
- **Global fix for local cohort:** other modes or stages regress. Segment and choose a lower-level lever.
- **Uncontrolled live change:** population or content shifts confound the result. Use a comparison strategy and record concurrent changes.
- **No rollback condition:** a harmful patch persists through debate. Predefine thresholds and preserve prior data.

## Verification

- Simulation question, model, policy, horizon, and omissions are explicit.
- Seeds and versions reproduce selected traces.
- Simple cases agree with independent calculations.
- Invariants run during every simulation step.
- Run count is justified by convergence or decision tolerance.
- Results include distributions, uncertainty, and sensitivity reversals.
- Telemetry begins from a decision and records its denominator.
- Events include schema, build, and balance-data versions.
- Instrumentation is reconciled against controlled sessions.
- Analysis handles cohorts, exposure, outliers, and missing data.
- Tuning changes identify a causal lever, expected movement, guardrails, and rollback.
- The final decision updates the source of truth and tuning log.
