---
name: godot-game-architecture
description: "Godot 4.x game architecture, authoritative state, data ownership, managers, autoloads, scene flow, and save/load boundaries. Use when deciding where runtime state lives, splitting scene-local and persistent systems, coordinating menus and gameplay scenes, designing new-run/reset behavior, or implementing versioned persistence."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Godot Game Architecture

Give every piece of state one authoritative owner at the narrowest lifetime that satisfies the game flow.

## Scope

Use this skill for:

- Classifying state by lifetime and choosing its authoritative writer.
- Separating static definitions, live simulation state, presentation, and persisted snapshots.
- Deciding whether a coordinator belongs in a scene or an autoload.
- Defining manager responsibilities and public mutation boundaries.
- Coordinating menu, gameplay, results, reload, and overlay scene flow.
- Designing new-session, reset, checkpoint, save, load, migration, and corruption behavior.
- Preventing scene transitions from leaking nodes or retaining stale references.

Do not use this skill for basic node selection, scene instantiation, custom-resource syntax, Input Map setup, or editor navigation; use the fundamentals skill. Do not use it to teach SOLID, signal mechanics, commands, or pooling; use the programming-patterns skill. Architecture may select an event or component boundary, but the patterns skill owns its implementation details.

## Inspect Before Designing

1. Locate `project.godot` and inspect `config/features` for the exact Godot 4.x target.
2. Read `[application]` to identify the main scene and `[autoload]` to inventory process-lifetime nodes and their order.
3. Inspect the project's scene, script, resource, and data layout. Preserve naming, folder, typing, and dependency conventions.
4. Trace every current scene transition and overlay. Record who initiates it, what is freed, what persists, and what the next scene expects.
5. Build a state inventory. For each value, record its writer, readers, lifetime, reset rule, definition source, and whether it is persisted.
6. Search for direct writes to shared fields, global lookups, `get_tree().root.add_child()`, manager references, user-file access, and startup emissions.
7. Reproduce baseline flows: fresh boot, new game, scene change, retry, return to menu, and save/load when present.
8. Inspect existing tests and any save fixtures before changing schemas or ownership.

Do not add a new manager until existing owners and autoloads have been evaluated.

## Classify State

Use four separate concepts:

- **Definition data:** authored facts shared by instances, often resources, such as item cost or unit base speed.
- **Runtime state:** mutable facts for this entity, room, match, or session, such as current health or discovered rooms.
- **Presentation state:** transient view details, such as selected tab, animation progress, or a visible tooltip.
- **Snapshot data:** primitive, versioned values crossing the persistence boundary.

Never use one shared resource as all four. In particular, mutating a definition resource to store runtime progress can change every consumer and complicate saves.

## Assign Ownership by Lifetime

Choose the nearest stable owner:

- Entity state belongs to the entity or one of its focused components.
- Room or level state belongs to that room/level coordinator.
- Match state belongs to the match root or a match model owned by it.
- Run state that crosses level scenes belongs to a session owner that survives those transitions.
- Profile progress and settings belong to profile/settings services with explicit persistence APIs.
- UI displays state but should not become the authoritative gameplay store.

For each state value, allow one writer. Other systems request mutations through methods and observe results through reads or events. Avoid public dictionaries that every scene edits directly.

## Decide Manager Lifetime

Keep a manager scene-local when its state should disappear with that scene. A level director, encounter tracker, or match referee normally belongs under the gameplay root.

Use an autoload only when at least one condition is true:

- The capability must exist before and after current-scene replacement.
- Multiple unrelated root scenes need the same process-lifetime service.
- It coordinates scene replacement itself.
- It owns session/profile state whose defined lifetime crosses those scenes.

An autoload must not retain ordinary nodes from a scene after that scene exits. Store plain state, stable IDs, or reacquired references. Provide explicit `start_new_*`, `reset`, and `restore` methods; process lifetime is not a reset policy.

## Design Scene Flow

- Give one coordinator authority to replace the current scene.
- Express destinations as exported `PackedScene` values or a centralized route map, not repeated string paths across UI and actors.
- Guard duplicate transitions caused by several inputs or end conditions in one frame.
- Treat replacement and overlay as different operations. Replace mutually exclusive roots; add pause, dialog, and loading overlays under a deliberate owner.
- Keep runtime-created gameplay nodes under the current scene so replacement frees them.
- Wait for `SceneTree.scene_changed` when initialization requires nodes in the incoming scene.
- Pull initial state after the scene is ready; do not rely on a global startup event that late listeners can miss.
- Define whether death, retry, next level, and return to menu preserve or reset run state.

## Define Persistence Boundaries

- Save snapshots, not nodes, callables, signal connections, or live resource references.
- Convert game objects to primitive dictionaries/arrays with stable IDs.
- Store user data under `user://`, never under imported `res://` content.
- Include a schema version and define migration or rejection for every supported old version.
- Validate types, ranges, IDs, and required relationships before mutating live state.
- Restore into a clean model first, then build or update scenes from that model.
- Keep disk I/O and serialization outside gameplay entities.
- Define save timing and failure behavior; do not let every subsystem write independently.

## Implementation Workflow

1. Write an ownership map for only the affected state and flows.
2. Define the narrow public API of the authoritative owner.
3. Route all mutations through that owner while preserving current behavior.
4. Make views initialize from a current snapshot and then observe later changes.
5. Put coordinators at the lifetime they manage; avoid promoting scene-local state to an autoload for convenience.
6. Centralize scene replacement and define reset semantics for every route.
7. Add snapshot conversion, validation, versioning, and storage as separate steps.
8. Remove obsolete duplicate state only after all readers use the authority.
9. Test lifecycle and persistence matrices, not only the happy path.

## Verification

Run existing project checks and exercise these cases where relevant:

- Fresh boot with no user data.
- Direct main-scene run and normal boot through the configured entry scene.
- New game after a previous run in the same process.
- Repeated menu, gameplay, result, retry, and next-level transitions.
- Two transition requests in the same frame.
- Scene exit while timers, awaits, or runtime-created nodes are active.
- UI created after state already changed; it must still show the current value.
- Save then load round trip into a clean session.
- Missing, truncated, malformed, unknown-version, and partially valid save data.
- Stable-ID resolution when authored content was renamed or removed.
- Profile/settings persistence without accidental run-state persistence.
- Remote scene tree and object counts after several loops; no old roots or scene references remain.

Report which state owner, reset policy, transition flow, and schema version were verified.

## Failure Modes

- **Everything becomes an autoload:** convenience replaced ownership. Move state to the narrowest scene or model lifetime.
- **Manager is a global bag:** unrelated systems write fields directly. Split coherent capabilities and expose mutation methods.
- **UI is stale on entry:** it waited for a past event. Read current state when initialized, then subscribe for changes.
- **New game keeps old values:** process-lifetime state has no explicit reset path.
- **Scene change leaks actors:** nodes were attached above `current_scene`, or a long-lived object retains them.
- **Autoload calls missing HUD nodes:** global initialization ran before the destination scene was ready. React after `scene_changed` or let the scene pull state.
- **Two owners disagree:** state is duplicated in UI, manager, entity, or resource. Choose one authority and derive the rest.
- **Shared definitions mutate:** a `.tres` used as a template also holds per-run progress. Split definition and runtime models.
- **Load half-applies:** validation and mutation were interleaved. Parse and validate a complete temporary snapshot first.
- **Old saves crash:** no schema version, migration, defaults, or unknown-ID policy exists.
- **Transition strings drift:** paths are scattered across buttons and actors. Centralize routing or export scene resources.

## References

- [State and data ownership](references/state-and-data-ownership.md)
- [Managers and scene flow](references/managers-and-scene-flow.md)
- [Persistence boundaries](references/persistence-boundaries.md)
