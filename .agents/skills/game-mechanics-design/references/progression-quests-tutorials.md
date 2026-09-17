# Progression, Quests, and Tutorials

Use this reference after the core loop is playable. These systems should expose new decisions, organize goals, and support learning rather than compensate for weak moment-to-moment play.

## Progression Layers

Separate four forms of progression:

- **Player learning:** The person gains knowledge or execution skill.
- **Avatar capability:** The controlled character or team gains power or options.
- **Content access:** New spaces, encounters, modes, or story states become available.
- **Expression and status:** Builds, cosmetics, records, collections, or social roles communicate identity and accomplishment.

A game can use one layer without the others. Choose deliberately. Numerical power can hide whether the player has learned, while a purely skill-based game may still need visible milestones so improvement is legible.

### Progression Design Procedure

1. List the decisions available in the first complete loop.
2. Define what the player should understand before a new option appears.
3. Add an unlock that changes a decision, relationship, or strategy.
4. Create a low-risk place to try it.
5. Revisit an earlier situation where the new option creates a different solution.
6. Check whether the old option remains useful in some context.

Prefer horizontal growth when possible: new verbs, combinations, roles, or tradeoffs. Vertical growth is appropriate when increased power communicates earned scale, opens previously impractical targets, or changes pacing. Avoid upgrades that only preserve the same time-to-defeat against proportionally larger numbers.

### Progression Curves

Document each curve with units:

- Cost per unlock.
- Expected earnings per minute or encounter.
- Expected time between decisions.
- Power increase relative to old and upcoming challenges.
- Number of meaningful choices before a cap.

Calculate expected time explicitly:

`time_to_unlock = remaining_cost / expected_net_gain_per_minute`

Test the first, middle, and final intervals. A formula that looks smooth can still create a long, empty stretch after bonuses, retries, or optional content are included.

### Progression Edge Cases

- A player saves points indefinitely and skips the intended learning sequence.
- A respec creates an invalid equipped loadout.
- An unlock arrives while the inventory or ability bar is full.
- A late-joining co-op player lacks a required traversal ability.
- New Game Plus carries keys or quest flags into incompatible content.
- A capped player keeps receiving unusable progression currency.
- A difficulty change alters rewards and creates a farming exploit.

## Quest Design

A quest is a structured promise: perform meaningful activity under stated conditions, then receive a result the game can reliably deliver. Its fiction, objectives, and system consequences should agree.

### Quest Frame

- **Purpose:** What experience, relationship, rule, or location does this quest develop?
- **Hook:** Why does the goal matter now?
- **Player activity:** Which existing actions carry the quest?
- **Complication:** What changes the plan rather than merely increasing quantity?
- **Choice:** Where can the player select method, priority, allegiance, route, or sacrifice?
- **World response:** What changes beyond a reward popup?
- **Reward:** What supports the next decision or resolves the promise?
- **Duration:** Expected active play, travel, waiting, and replay after failure.

Gathering, defeating, delivering, escorting, and conversing are implementation shapes, not complete designs. A good quest uses a shape to develop the game's core activity. Change context, constraints, relationships, or consequences instead of only replacing the target name and count.

### Quest State Model

Define explicit states and legal transitions. A typical model may include:

`unavailable -> offered -> active -> ready_to_turn_in -> completed`

Add `failed`, `abandoned`, or `expired` only when the design needs them. For every transition, specify:

- Trigger and authority.
- Persistent data written.
- Signals or UI updates emitted.
- Rewards granted exactly once.
- Cleanup of spawned targets and markers.
- Behavior after save/load or multiplayer reconnection.

Keep objective progress idempotent where events may be delivered twice. Use stable identifiers rather than node paths for persistent quest targets.

### Quest Verification

Unit-test:

- Every legal and illegal transition.
- Duplicate completion events and reward idempotency.
- Progress before acceptance when retroactive credit is or is not intended.
- Save/load in each state.
- Target despawn, scene change, player death, abandonment, and expiry.
- Shared credit, late join, simultaneous completion, and host migration if applicable.

Playtest:

- Whether players can restate the goal without reading the full log.
- Whether travel and waiting dominate active decisions.
- Whether optional quests drown out the main objective.
- Whether the reward is useful at the level where it arrives.
- Whether a choice produces a visible world or relationship response.

## Tutorial Design

Treat onboarding as a sequence of playable proofs, not a front-loaded manual.

For each required concept, use this cycle:

1. **Invite:** Frame an obvious opportunity to try the action.
2. **Protect:** Remove unrelated threats and severe failure costs.
3. **Respond:** Acknowledge success, failure, and attempted-but-invalid input.
4. **Require:** Place one fair obstacle that needs the concept.
5. **Vary:** Change one condition so the player demonstrates understanding.
6. **Recall:** Ask for it later without a prompt.
7. **Combine:** Pair it with another established skill.

Prompt only when behavior shows a prompt is needed. Dismiss prompts after demonstrated understanding, not merely after a timer. If the player already performs the action, credit it immediately.

### Input and Accessibility

- Display the current Input Map binding, never a hard-coded key name.
- Update prompts when the active device changes.
- Support remapping, hold/toggle alternatives, timing assists, and reduced simultaneous-input demands where relevant.
- Never communicate a required rule through color, sound, fine motion, or text alone.
- Allow tutorial replay and provide a skip path that initializes all required state correctly.
- Keep modal messages away from timing-critical play unless pausing is intentional and reliable.

### Tutorial Edge Cases

- The player reaches the teaching area from an unintended direction.
- The required object was consumed, destroyed, or collected early.
- A prompt repeats after reload or on every scene entry.
- The player uses an alternate valid action the tutorial does not recognize.
- A controller disconnects during a hold or chord instruction.
- Localization expands text over the action area.
- Slow motion or pause changes the timer used by the lesson.
- Co-op partners complete the demonstration for one another.

## Combined Verification Plan

Create a debug route that can start at every progression tier, quest state, and tutorial step. Add deterministic fixtures for unlock costs, objective events, and rewards. Visually inspect the path with prompts disabled, alternate input bindings, high UI scale, color-vision simulation, and audio muted.

The combined systems are healthy when progression introduces useful decisions, quests give those decisions purpose, and tutorials let players discover the rules with minimal interruption.
