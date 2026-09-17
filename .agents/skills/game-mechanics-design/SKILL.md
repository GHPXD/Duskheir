---
name: game-mechanics-design
description: "Selects and prototypes player-facing mechanics for meaningful choices, challenge, progression, quests, combat, collection, rewards, and tutorials. Use for deciding what the player repeatedly does and how that action supports the intended experience; defer numerical tuning to game-balance and cross-system feedback analysis to game-systems-design."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Game Mechanics Design

Use this skill to turn a desired player experience into a small, coherent, testable set of mechanics. Do not begin with a genre feature checklist. Begin with the player's repeated decisions and the evidence needed to determine whether those decisions produce the intended experience.

## Core Principles

- Treat player motivations as hypotheses, not fixed personality labels.
- Prefer a few mechanics with strong interactions over many isolated features.
- Make important options legible, genuinely distinct, and consequential.
- Build challenge from learned skills and understandable rules, not hidden exceptions.
- Add progression, rewards, and currencies only when they strengthen the play itself.
- Teach by arranging safe opportunities to act, observe, retry, and later combine skills.
- Protect player time. Failure should teach, and repetition should have a clear purpose.

## Workflow

### 1. Inspect the Current Game

Before proposing changes:

- Confirm the Godot version, target devices, input methods, audience, session length, and multiplayer assumptions.
- Read the project entry scene, gameplay scenes, input map, state-owning scripts, resources, save model, and existing tests.
- Play the smallest runnable loop when possible. Record what the player can do, what changes, how success and failure occur, and what feedback is visible.
- Separate shipped constraints from ideas that are merely mentioned in notes or unused code.

If no project exists, ask for or state explicit assumptions instead of inventing production constraints.

### 2. Write the Experience Target

Express the target in one sentence:

`The player should feel <experience> by repeatedly <activity> while deciding <tradeoff>.`

Then identify:

- Primary motivation hypotheses, such as mastery, discovery, expression, completion, social connection, narrative curiosity, or relaxation.
- The intended emotional rhythm, including pressure, relief, surprise, and reflection.
- The acceptable cognitive, motor, time, and failure burden.
- Ethical boundaries around monetization, randomness, loss, social pressure, and retention.

Use [Motivation, Choice, and Challenge](references/motivation-choice-challenge.md) to develop and test this target.

### 3. Map the Core Loop

Write the shortest repeatable loop as:

`observe -> choose -> act -> feedback -> changed state -> next decision`

For each step, record:

- Information available to the player.
- Player action and input burden.
- Rules and resources consulted.
- Immediate and delayed consequences.
- Audiovisual feedback.
- How the next decision differs from the previous one.

If two consecutive steps do not change what the player understands or can decide, combine or remove one.

### 4. Select Mechanics Deliberately

Generate several candidates, then score their fit against the experience target, platform, scope, accessibility, interaction depth, tutorial cost, content burden, and technical risk. Reject a mechanic that fails a non-negotiable constraint even if it is exciting in isolation.

Use [Mechanic Selection Workshop](references/mechanic-selection-workshop.md) for the selection matrix, mechanic specification, and vertical-slice procedure.

### 5. Shape Choice and Challenge

For each important decision:

- Ensure at least two options are viable in the current context.
- State what the player knows before choosing.
- Give each option a cost, opportunity cost, risk, or commitment.
- Make the consequence observable at an appropriate time.
- Decide whether the choice is reversible and communicate that honestly.

Build challenge by varying timing, space, resource pressure, uncertainty, coordination, or planning. Change one demand at a time when teaching; combine known demands later. Provide recovery paths where a single small mistake should not decide the entire run.

### 6. Add Supporting Systems

Only after the core interaction works, evaluate:

- Progression and unlocks.
- Quests and objective structure.
- Combat encounters and enemy roles.
- Collection and inventory constraints.
- Resource creation, conversion, storage, and spending.
- Tutorial prompts and practice spaces.

Use [Progression, Quests, and Tutorials](references/progression-quests-tutorials.md) and [Combat, Collection, and Economies](references/combat-collection-economy.md). Trace every supporting system back to the core loop. Remove rewards that motivate bypassing the intended play.

### 7. Prototype the Riskiest Question

Create the smallest playable test that can disprove the design. Use placeholder presentation, deterministic scenarios, and exposed tuning values. Do not build a full progression tree or content catalog to test whether one combat decision is interesting.

Capture expected observations before testing, for example:

- Players can explain the tradeoff without designer help.
- At least two strategies succeed for different reasons.
- A first failure reveals information used on the next attempt.
- Players notice a state change through normal play, not a debug display.

### 8. Iterate from Evidence

Observe behavior before asking for opinions. Record confusion, hesitation, repeated choices, accidental inputs, unused systems, recovery behavior, and where players stop. Change one causal variable per test when practical. Keep a short decision log linking each revision to evidence.

## Verification

### Design Checks

- Every mechanic supports the experience target or a required constraint.
- Important choices have distinct consequences and no permanently dominant option.
- Challenge relies on knowledge or skills the game has already made learnable.
- Rewards do not erase the tradeoffs they are meant to support.
- Progression changes decisions rather than only increasing numbers.
- Tutorials can be skipped, replayed, and completed with remapped inputs.

### Implementation Checks

- Keep tuning data separate from irreversible state transitions where the project permits it.
- Unit-test economy transactions, quest transitions, unlock prerequisites, reward bounds, and deterministic random tables.
- Test save/load during every long-lived state, including mid-quest, full inventory, and partially spent rewards.
- Run visual tests for telegraphs, feedback timing, HUD readability, color-independent cues, and camera framing.
- Test at low and high frame rates, with keyboard, controller, and touch when supported.

### Playtest Edge Cases

- Novice and expert players reach different challenge limits.
- Players ignore the intended reward or optimize away the intended activity.
- Inventory is full when a mandatory reward arrives.
- A quest target is unavailable, already defeated, duplicated, or removed on reload.
- Random rewards produce long unlucky streaks or duplicate-only outcomes.
- Currency overflows, becomes negative, or can be duplicated through retries or disconnects.
- Tutorial conditions trigger after the player has already learned or rebound the action.
- Co-op players join, leave, or complete an objective out of order.

## Expected Output

Produce a compact design artifact containing:

1. Experience target and constraints.
2. Core loop and major decisions.
3. Selected mechanics with rejected alternatives and reasons.
4. Progression, quest, combat, collection, economy, and tutorial implications that actually apply.
5. Prototype scope, tuning variables, edge cases, and verification plan.
6. Open assumptions clearly marked for validation.
