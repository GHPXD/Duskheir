# Managers and Scene Flow

## Place Coordinators by Lifetime

Use a scene-local coordinator when it manages nodes owned by that scene:

```text
Match
|-- MatchReferee
|-- Units
|-- Arena
`-- HUD
```

When `Match` leaves, its referee, units, and connections leave together.

Use an autoload for a process-lifetime capability such as a scene router, settings service, or session model that intentionally survives root replacement. Inspect existing `[autoload]` entries first; extending one coherent service is often better than adding another global.

## Autoload Review Checklist

For each autoload, record:

- Its single responsibility.
- Why its lifetime must exceed `current_scene`.
- Public methods and mutable fields.
- Startup order dependencies on other autoloads.
- Explicit reset behavior.
- References it may hold.
- What happens when the current scene changes.
- Whether it performs disk I/O, routing, state ownership, or presentation and should be split.

Autoloads are nodes in the root tree, not magical static classes. They process, receive notifications, and retain state until explicitly reset or the process exits.

## Centralize Root Replacement

A small router can serialize scene changes and report errors. Configure it as an autoload only if the project needs process-lifetime routing:

```gdscript
extends Node

signal transition_started
signal transition_finished(scene: Node)

var _busy: bool = false

func replace_with(next_scene: PackedScene) -> int:
	if _busy:
		return ERR_BUSY
	if next_scene == null:
		return ERR_INVALID_PARAMETER

	_busy = true
	transition_started.emit()
	var error := get_tree().change_scene_to_packed(next_scene)
	if error != OK:
		_busy = false
		return error

	await get_tree().scene_changed
	_busy = false
	transition_finished.emit(get_tree().current_scene)
	return OK
```

Call the coroutine with `await` when the caller needs completion:

```gdscript
var error := await SceneRouter.replace_with(game_scene)
if error != OK:
	push_error("Could not enter the game scene: %s" % error_string(error))
```

Match the project's actual router name and API. Avoid adding this abstraction when one local transition is sufficient.

## Route Semantics, Not Scattered Paths

Prefer one of these approaches:

- Export `PackedScene` destinations on a flow coordinator.
- Store a small route-to-`PackedScene` map in one router.
- Expose semantic methods such as `start_new_run()`, `retry_match()`, and `return_to_menu()` that apply reset policy before routing.

Do not repeat `change_scene_to_file("res://...")` in player death, buttons, portals, and match logic. Besides path drift, those call sites can disagree about state reset and transition guards.

## Keep Dynamic Nodes Under the Current Scene

Root replacement frees `current_scene` and its descendants. A node added directly under `get_tree().root` sits outside that boundary and can survive unexpectedly.

Use dedicated containers owned by the scene for generated rooms, actors, effects, and UI. If a system intentionally creates a root-level node, document who removes it and test several transitions.

Long-lived managers should not retain:

- Player, camera, HUD, room, or unit node references after scene exit.
- Await continuations that mutate a replaced scene.
- Callables bound to freed scene objects.
- Collections of scene-owned nodes that are never cleared.

Use IDs or plain models for persistent state and reacquire scene adapters after `scene_changed`.

## Coordinate Incoming Scene Initialization

Autoload `_ready()` runs before ordinary main-scene nodes. Therefore:

- Do not emit the only initial-state notification from an autoload `_ready()` and assume scene listeners receive it.
- Let incoming views pull current state during their own binding.
- Use `SceneTree.scene_changed` when a coordinator must act after root replacement.
- Keep destination-specific node setup in the destination root where possible.
- Avoid branching on a scene's display name; use an explicit route, type, group, or setup contract.

If a scene must support direct editor execution, give it safe defaults or an explicit development bootstrap rather than relying accidentally on a menu flow.

## Replace Versus Overlay

Replace the root for mutually exclusive high-level states such as menu and gameplay when their trees should not coexist.

Overlay under a deliberate owner for:

- Pause menus.
- Modal dialogs.
- Inventory screens that preserve the world.
- Transition fades.
- Loading presentation.

For overlays, define pause behavior, input consumption, focus restoration, and teardown. A pause overlay that leaves gameplay input active is not a complete boundary.

## Model the Game Loop

Write allowed transitions before wiring buttons:

```text
Boot -> Menu
Menu -> New Run -> Gameplay
Gameplay -> Results
Results -> Retry | Menu
Gameplay -> Pause -> Gameplay
```

For each arrow, specify:

- State created, retained, or reset.
- Save/checkpoint policy.
- Scene replaced or overlaid.
- Input enabled during transition.
- Error destination if loading fails.

This prevents individual screens from inventing conflicting flow rules.

## Scene-Flow Verification

- Trigger every allowed transition repeatedly.
- Trigger forbidden and duplicate transitions; they should fail predictably.
- Watch `current_scene` and the Remote tree after each replacement.
- Compare node/object counts after several complete loops.
- Start the gameplay scene directly if the team supports that workflow.
- Exit during a deferred callback or await and confirm no stale scene mutation occurs.
- Confirm a new run resets session state while a next-level transition preserves only intended fields.
- Confirm overlays consume input and restore focus/process state on close.
