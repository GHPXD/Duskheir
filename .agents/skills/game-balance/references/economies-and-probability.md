# Economies and Probability

Use this reference for resource-flow models, affordability and stability checks, random outcome tables, expected value, variance, and streak analysis.

## Economy Boundary

First decide whose economy is being modeled:

- One player during an encounter.
- One player across progression.
- A cohort over calendar time.
- A shared market with transfers between players.
- The entire game, including resources created or destroyed by system actors.

A transfer is a sink for one actor and a source for another, but it is neither at the whole-economy boundary unless a fee destroys part of it.

## Resource Ledger

Create one row per resource or currency.

| Field | Prompt |
| --- | --- |
| Resource | What is counted, exchanged, or constrained? |
| Purpose | Which decisions does it enable? |
| Sources | How much enters, how often, and for whom? |
| Sinks | How much leaves, how often, and for whom? |
| Transfers | Which actors exchange it without creating or destroying it? |
| Conversions | Which resources become which others, at what rate and loss? |
| Stock | Where is it stored? |
| Bounds | Capacity, debt floor, decay, or expiration |
| Visibility | Exact, estimated, or hidden from players |
| Time scale | Action, session, day, season, or lifetime |

## Stock and Flow Equations

For one actor over one interval:

```text
stock_next = stock_now
             + created
             + transfers_in
             - destroyed
             - transfers_out
```

For a currency with purchases and fees:

```text
stock_next = stock_now + earned + granted + purchased
             - prices_paid - fees - decay
```

At the whole-economy boundary, player-to-player prices paid cancel with receipts. Transaction fees do not.

Useful rates:

```text
net_flow_per_hour = source_per_hour - sink_per_hour
source_sink_ratio = source_per_hour / max(sink_per_hour, epsilon)
currency_velocity = amount_spent_during_period / average_stock_during_period
```

Define affordability piecewise:

```text
time_to_afford = 0                                      when current_stock >= price
time_to_afford = (price - current_stock) / net_flow     when net_flow > 0
time_to_afford = unreachable                            otherwise
```

Interpret ratios in context. A progression economy may intentionally have positive net flow; a mature shared economy may need stronger recurring sinks.

## Economy Scenario Sheet

| Scenario | Starting stock | Source events | Sink events | Ending stock | Key purchase affordable? | Decisions remaining? |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| first session | | | | | | |
| typical returning player | | | | | | |
| expert optimizer | | | | | | |
| low-engagement player | | | | | | |
| hoarder | | | | | | |
| spender | | | | | | |
| maximum progression | | | | | | |

Run the ledger by cohort and percentile. Average stock can look healthy while new players are starved and veterans hold unusable fortunes.

## Price and Reward Checks

For every important purchase or reward, calculate:

- Earliest, median, and late time-to-afford.
- Opportunity cost in units of the next-best purchase.
- Purchase frequency and repeatability.
- Value after progression, not only at introduction.
- Whether saving is always superior to spending now.
- Whether the item creates new decisions or only larger output.
- Whether the price is a meaningful sink at the cohort that can buy it.

For a conversion chain, multiply yields and losses:

```text
final_output = initial_input * yield_1 * yield_2 * ... * yield_n
```

Track time, capacity, and byproducts separately. A profitable ratio can still stagnate if conversion is too slow or storage is too small.

## Worked Economy Example: Beacon Marks

A route game grants `12` marks after an ordinary expedition. Expected repairs cost `3` marks per expedition. A route-chart upgrade costs `45` marks.

```text
net_marks_per_expedition = 12 - 3 = 9
expeditions_to_upgrade = 45 / 9 = 5
```

If the target is an upgrade every 4 to 6 ordinary expeditions, the mean fits. That is not enough verification:

- A damaged expedition may cost `11` to repair, creating only `1` net mark.
- A flawless expedition may avoid repairs and grant a `6` mark bonus.
- Players may also spend `8` marks on a temporary forecast.
- A second upgrade at `90` marks changes cumulative pacing.

Model at least the 10th, 50th, and 90th percentile time-to-upgrade and the share of expeditions where the forecast is a credible alternative. If players never buy forecasts, raising their reward may be less appropriate than changing forecast timing, information value, or duration.

## Economy Diagnostics

### Inflation

Signals:

- Holdings grow faster than useful prices or sinks.
- Currency velocity falls while balances rise.
- Formerly meaningful rewards become negligible.
- Players substitute another scarce item as an informal store of value.

Checks:

- Source and sink rates by cohort and version.
- Holdings at median, 90th, and 99th percentiles.
- Share of stock owned by top holders.
- Repeatable versus one-time sinks.
- Resource creation from exploits, automation, or duplicate grants.

### Stagnation

Signals:

- Players delay every purchase.
- Transactions fall despite unmet goals.
- Mandatory maintenance consumes most income.
- One failure removes the means to recover.

Checks:

- Net flow after mandatory costs.
- Time-to-afford by low and median performance.
- Expected value of spending now versus saving.
- Minimum viable action budget.
- Availability of costly but credible recovery routes.

### Hoarding

Hoarding can indicate uncertainty, missing future information, weak sinks, excessive loss aversion, or inventory friction. Do not assume the stock cap alone is the solution. Test whether players understand future needs and whether current spending produces visible value.

## Probability Event Worksheet

```text
Event name:
Player decision affected:
Possible outcomes and values:
Weights or probabilities:
Independent, conditional, or stateful:
Replacement/reset rule:
Frequency:
Impact of one outcome:
Player-visible odds:
Expected value:
Variance/standard deviation:
Failure or drought percentiles:
Guarantee, pity, or cap state:
Rounding and random-seed behavior:
```

## Core Probability Formulas

For equally likely outcomes:

```text
P(success) = successful_outcomes / total_outcomes
```

For weighted outcomes:

```text
P(outcome_i) = weight_i / sum(all_weights)
```

Expected value:

```text
E[X] = sum(P(outcome_i) * value_i)
```

Variance and standard deviation:

```text
Var(X) = sum(P(outcome_i) * value_i^2) - E[X]^2
SD(X) = sqrt(Var(X))
```

For independent events:

```text
P(A and B) = P(A) * P(B)
P(A or B) = P(A) + P(B) - P(A and B)
```

For a conditional event:

```text
P(A and B) = P(A) * P(B given A)
```

For at least one success in `n` independent trials with constant chance `p`:

```text
P(at_least_one) = 1 - (1 - p)^n
P(no_successes) = (1 - p)^n
```

Expected attempts to first success for independent constant chance:

```text
E[attempts] = 1 / p
```

Expected attempts is not a guarantee or a typical maximum. Always inspect percentiles.

Trials needed to reach probability `q` of at least one success:

```text
n = ceil(ln(1 - q) / ln(1 - p))
```

This form requires `0 < p < 1` and `0 < q < 1`.

For exactly `k` successes in `n` independent trials:

```text
P(K = k) = combination(n, k) * p^k * (1 - p)^(n - k)
```

## Without Replacement

When outcomes are removed from a pool, probabilities change after every draw.

If a pool begins with `S` successes among `T` items, the chance of two successes without replacement is:

```text
P(two successes) = (S / T) * ((S - 1) / (T - 1))
```

Do not use independent-roll formulas for decks, bags, rotating shops, or loot pools unless the item is replaced and the pool resets exactly as assumed.

## Stateful Guarantees

A guarantee, escalating chance, duplicate protection, or cooldown makes each roll conditional on prior state.

Model it as a transition table:

| State before roll | Success chance | On failure | On success |
| --- | ---: | --- | --- |
| 0 prior failures | | state 1 | reset or new state |
| 1 prior failure | | state 2 | reset or new state |
| ... | | | |

Calculate or simulate the complete distribution. Do not advertise the base chance as if it described the whole stateful process. Preserve any required disclosures and avoid misleading player-facing odds.

## Worked Probability Example: Salvage Cache

| Outcome | Weight | Probability | Value |
| --- | ---: | ---: | ---: |
| Scrap | 60 | 0.60 | 2 |
| Battery | 25 | 0.25 | 6 |
| Lens | 10 | 0.10 | 15 |
| Core | 5 | 0.05 | 40 |

Expected value:

```text
E[X] = 0.60*2 + 0.25*6 + 0.10*15 + 0.05*40 = 6.2
```

Second moment and spread:

```text
E[X^2] = 0.60*2^2 + 0.25*6^2 + 0.10*15^2 + 0.05*40^2 = 113.9
Var(X) = 113.9 - 6.2^2 = 75.46
SD(X) ~= 8.69
```

The standard deviation exceeds the mean, so short sequences feel volatile.

Chance of at least one Core in 20 independent caches:

```text
1 - (1 - 0.05)^20 ~= 0.642
```

About 35.8% of players would still see no Core after 20 caches under the independent model. If that drought is unacceptable, change the distribution or add a stateful guarantee, then recalculate the economy impact and full wait-time distribution.

## Randomness Diagnostics

- **Mean is correct, experience is harsh:** inspect variance, lower percentiles, droughts, and impact per event.
- **Players distrust displayed odds:** audit implementation, rounding, dependency, event selection, and whether the displayed chance is conditional.
- **Rare reward floods economy:** multiply probability by event frequency, active population, and reward value.
- **Randomness overwhelms skill:** reduce impact per event, give players decisions after the roll, or distribute randomness across more lower-impact events.
- **Outcome feels scripted:** check streak prevention, small sample size, visible patterns, and repeated seed behavior.
- **Guarantee is exploitable:** inspect reset conditions, shared counters, mode switching, and partial progress persistence.
- **Choice is fake under uncertainty:** compare expected value and downside for each option using the information actually available to the player.

## Verification

- Economy boundary and time unit are explicit.
- Transfers are not mistaken for creation or destruction at the aggregate level.
- Every stock reconciles from one interval to the next.
- Affordability is checked by cohort and percentile.
- Weighted probabilities normalize to 1 within tolerance.
- Analytical results match enumeration for a small test case.
- Independent and dependent events are distinguished correctly.
- Expected value, variance, and relevant tail probabilities are reported together.
- Stateful guarantees are modeled with their reset and transition rules.
- Random reward changes are propagated through source rates and economy forecasts.
- Player-visible odds and implementation behavior agree.
