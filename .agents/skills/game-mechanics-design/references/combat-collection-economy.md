# Combat, Collection, and Economies

Use this reference to design interacting action and resource systems. Model them together: combat often creates items and currency, collection changes combat capability, and the economy determines which tactics remain affordable.

## Combat as Decisions

Start with the decisions, not the weapon list. A combat loop can ask the player to manage:

- Position and distance.
- Timing and commitment windows.
- Target priority.
- Attack, defense, interruption, and escape.
- Limited ammunition, stamina, cooldowns, health, or attention.
- Information such as intent, vulnerability, and unseen threats.
- Cooperation, role coverage, or contested space.

For every attack, specify startup, active, recovery, movement allowed, targeting rule, cost, effect, counterplay, and feedback. A high-damage attack is not interesting by itself; its commitment and counterplay create the decision.

### Enemy Role Card

- Role in the encounter:
- Pressure applied:
- Range and movement pattern:
- Readable intent cue:
- Player responses that work:
- Response that is risky but rewarding:
- Interaction with another enemy role:
- Recovery opportunity after success:
- Failure information communicated:

Design roles that combine into new priorities. Two enemies that differ only in health create quantity, not a new tactical relationship. Introduce roles separately before combining them under pressure.

### Combat Fairness

- Telegraph dangerous actions early enough for the intended response and input latency.
- Keep the cue visible and audible under expected effects, camera angles, and crowds.
- Make hit geometry agree with presentation.
- Distinguish invulnerability, armor, interruption resistance, and missed attacks through feedback.
- Preserve a recovery option unless one-error failure is the explicit tested premise.
- Avoid off-camera damage without directional warning and a reasonable response.
- Ensure adaptive difficulty does not invalidate learned timings or make success feel arbitrary.

### Combat Verification

Unit-test damage ordering, invulnerability windows, resource costs, cooldown boundaries, faction filters, death idempotency, and reward ownership. Test simultaneous hits, healing at zero health, attack cancellation, pause, slow motion, variable frame rate, and save/load during persistent encounters.

Visually test telegraphs at minimum and maximum camera distance, with effects stacked, animation speed changed, audio muted, and color distinctions removed. Record input-to-feedback latency and collision shapes during slow-motion debug playback.

## Collection with Purpose

Every collectible needs a role beyond increasing a counter. It may:

- Open a route or reveal information.
- Enable a build or temporary tactic.
- Feed crafting, trading, or progression.
- Mark exploration or completion.
- Support expression, lore, or social display.
- Create route-planning risk through placement.

Define the complete lifecycle:

`spawn or reveal -> notice -> reach -> acquire -> store -> inspect -> use, trade, transform, or complete -> remove or persist`

Check every transition for feedback and persistence.

### Collection Design Questions

- Can players distinguish required, valuable, dangerous, and decorative items without opening a menu?
- Does placement create an interesting route or merely cleanup work?
- Are duplicates useful, convertible, protected, or frustrating?
- Is completion status visible before the final missing item?
- Can an item be permanently missed, and is that communicated?
- Does inventory capacity create decisions or only menu maintenance?
- Can automatic pickup, hold-to-confirm, or magnet range reduce unnecessary motor burden?

### Collection Edge Cases

- Inventory is full during a mandatory pickup.
- Two players collect the same networked item simultaneously.
- Pickup animation plays but persistence fails.
- An item falls outside navigation or collision bounds.
- A unique item is sold, dropped, consumed, or duplicated.
- A collectible is acquired before its quest starts.
- Procedural generation creates an unreachable or absent final item.
- A save from an older content version references a removed item definition.

## Economy as a Flow Network

Model each resource with explicit units and nodes:

- **Faucet:** Creates the resource.
- **Sink:** Permanently consumes it.
- **Converter:** Exchanges it for another resource or capability.
- **Store:** Holds it, possibly with a cap or decay.
- **Gate:** Requires an amount, rate, item, or state without necessarily consuming it.
- **Transfer:** Moves ownership between players or agents.

Draw arrows between nodes and label each with amount, cadence, prerequisites, and uncertainty. Include time and player attention as resources even if the UI does not show them.

### Basic Measures

Use rates with named units:

`net_currency_per_minute = expected_income_per_minute - expected_spend_per_minute`

```text
if price <= current_balance: minutes_to_purchase = 0
elif net_currency_per_minute <= 0: minutes_to_purchase = unreachable
else: minutes_to_purchase = (price - current_balance) / net_currency_per_minute
```

`expected_drop_value = sum(drop_probability * item_value)`

Do not balance only around averages. Inspect variance, guaranteed rewards, unlucky streaks, player skill, route efficiency, and differences between first-time and repeat rewards.

### Economy Design Procedure

1. List every persistent and encounter-only resource.
2. Give each resource one clear primary purpose.
3. Inventory all faucets, sinks, converters, stores, gates, and transfers.
4. Estimate rates for novice, typical, expert, and exploitative play.
5. Plot balances over representative sessions and progression tiers.
6. Identify where abundance removes decisions and scarcity blocks experimentation.
7. Add caps, decay, taxes, repair, rerolls, or conversion only when each produces a desired choice.
8. Re-simulate after every reward, price, drop-rate, or progression change.

### Economy Failure Modes

- **Inflation:** Persistent income outpaces meaningful sinks, making prices irrelevant.
- **Starvation:** Required spending exceeds reliable income and blocks normal play.
- **Hoarding:** Future uncertainty makes spending feel unsafe, so upgrades go unused.
- **Dead currency:** A resource remains after all its uses are exhausted.
- **Circular profit:** A conversion loop returns more value than it consumes.
- **Dominant farm:** One low-risk activity outperforms the rest of the game.
- **Reward inversion:** Harder or longer content pays less per unit of effort with no other value.
- **Transaction exploit:** Retry, rollback, reconnect, or concurrency grants the same reward twice.

## Transaction and Simulation Tests

Automate these where possible:

- A transaction either applies all debits and credits or none.
- Balances respect documented lower and upper bounds.
- Purchases cannot race, duplicate, or spend stale balances.
- Rewards are granted once under repeated events.
- Random tables normalize correctly and honor guarantees.
- Conversion cycles cannot create unbounded value.
- Inventory stacking, splitting, moving, dropping, and save/load preserve total quantity.
- Economy simulations report percentiles and worst streaks across many deterministic seeds.

Run a visual end-to-end test from combat telegraph through defeat, drop, pickup, inventory display, sale, purchase, equipment change, and the resulting combat effect. This catches broken handoffs that isolated balance sheets miss.
