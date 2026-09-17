# Mechanic Selection Workshop

Use this reference to choose a mechanic set under real production constraints and turn the result into a falsifiable prototype.

## 1. Constraint Brief

Write the immovable facts first:

- Experience target:
- Intended audience and prior knowledge:
- Target platform and input methods:
- Typical and maximum session length:
- Single-player, local multiplayer, or network model:
- Accessibility requirements:
- Team, schedule, content, and technology limits:
- Business and ethical constraints:
- Existing systems that must remain compatible:

Do not score candidates until stakeholders agree which constraints are mandatory. A weighted score must not allow a mechanic to buy its way past a hard accessibility, safety, legal, or schedule limit.

## 2. Generate Candidate Mechanics

Generate mechanics from the target activity, not from genre convention. For each candidate, write a single state-changing rule:

`When <precondition>, the player may <action>, causing <state change> at <cost or risk>, communicated by <feedback>.`

Generate at least one candidate that:

- Deepens an existing action.
- Reuses an existing entity in a new relationship.
- Removes an action or resource.
- Changes information rather than power.
- Changes timing, space, or commitment.
- Supports an accessibility need without creating a separate lesser mode.

Combining and subtracting often produces more coherent designs than adding unrelated actions.

## 3. Apply Knockout Gates

Reject or redesign a candidate when any answer is no:

- Can the target input devices express it reliably?
- Can the intended audience perceive and understand its required cues?
- Can the team prototype its riskiest part within the time box?
- Can it coexist with save, networking, camera, level, and content constraints?
- Does it respect the project's ethical and monetization boundaries?
- Does it support the experience target rather than merely add novelty?

Record the rejection reason. A rejected idea may become useful if a constraint changes.

## 4. Score Relative Fit

Choose weights from 0 to 3 for the project, then score candidates from 0 to 5. Useful criteria include:

- Experience fit.
- Quality of repeated decisions.
- Interaction with current mechanics.
- Learnability and feedback clarity.
- Accessibility across supported inputs and senses.
- Prototype cost.
- Ongoing content burden.
- Technical and production risk.
- Tuning and testability.

Calculate:

`fit_score = sum(weight * score) / sum(weight * 5)`

The number supports discussion; it does not make the decision. Write one sentence explaining the strongest evidence and largest uncertainty for each finalist.

## 5. Check Interaction Density

Build a mechanic interaction map. Draw a link when two mechanics alter each other's decisions, costs, timing, information, or outcomes.

Prefer mechanics that:

- Interact with several existing rules in understandable ways.
- Create context-dependent choices rather than one universal answer.
- Can be introduced and developed through multiple situations.
- Produce useful feedback even when an attempted action fails.

Question a mechanic that needs a dedicated button, tutorial, content family, UI mode, and reward track but rarely affects other play. Its total cost is larger than its implementation estimate.

## 6. Write a Mechanic Specification

Use this compact format:

### Intent

- Player value served:
- Decision created:
- Intended emotional effect:

### Rule

- Inputs and preconditions:
- State read:
- State changed:
- Costs, cooldowns, and limits:
- Success, partial success, failure, and cancellation:
- Interaction with existing mechanics:

### Communication

- Anticipation cue:
- Action feedback:
- Result feedback:
- Persistent state display:
- Color-, audio-, text-, and motion-independent alternatives:

### Tuning

- Exposed variables with units:
- Safe minimum and maximum:
- Expected baseline:
- Relationships that should remain invariant:

### Persistence and Authority

- Saved state:
- Network authority if applicable:
- Duplicate-event protection:
- Migration concern:

### Verification

- Hypothesis that could be disproved:
- Instrumented observations:
- Unit tests:
- Visual tests:
- Edge cases:

## 7. Build a Question-Led Vertical Slice

The slice should answer one high-risk question, such as whether a cooldown creates target-priority decisions or merely waiting.

Include only:

- One representative environment.
- The minimum actions and entities needed for the decision.
- Placeholder but unambiguous cues.
- Exposed tuning values.
- A deterministic restart.
- Debug labels or traces that can be disabled for player tests.
- Event logging for decisions, failures, timings, and state transitions.

Exclude metagame progression, broad content catalogs, final art, and unrelated polish unless the hypothesis specifically concerns them.

## 8. Run the Test

Before each session, write expected evidence and a decision threshold. Example:

- Hypothesis: limited shield energy creates a choice between blocking now and preserving a safe retreat.
- Evidence: players sometimes accept a small hit to preserve energy and can explain why.
- Failure threshold: nearly every player holds block until empty in every encounter.
- Next experiment: make retreat consume shield energy, improve the low-energy cue, or replace the shared resource with a commitment window.

Observe first. Interview afterward with neutral prompts such as `What were you considering there?` Avoid explaining the mechanic before testing comprehension.

## 9. Decide and Document

Choose one outcome:

- Keep and expand.
- Keep but revise the rule.
- Run a targeted comparison.
- Defer because a prerequisite is unresolved.
- Cut because evidence contradicts the target or cost is unjustified.

Record the decision, evidence, known limitations, and next falsifiable question. Do not preserve a mechanic solely because implementation effort has already been spent.

## Final Verification

- Re-run the core loop without debug explanations.
- Test the mechanic alone, in combination, and under resource extremes.
- Test novice, expert, alternate-input, low-frame-rate, and accessibility-assisted play.
- Confirm that removing the mechanic measurably weakens the intended experience.
- Confirm that adding it does not make another mechanic irrelevant.
- Verify implementation state through death, restart, scene change, save/load, and network interruption when applicable.

A selected mechanic is ready for production only when its value is observed in play, its costs are understood, and its behavior can be verified at system boundaries.
