---
name: godot-2d-gameplay
description: "Builds and debugs Godot 4.x moment-to-moment 2D gameplay: CharacterBody2D movement, platformer or top-down physics, TileMapLayer worlds, animation, hitboxes, projectiles, health, combat feedback, cameras, and game feel. Use for player controllers, jumping, collisions, tilemaps, damage, shooting, and 2D polish; use godot-programming-patterns for pooling architecture."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Godot 2D Gameplay

Implement gameplay inside the project's existing scene, input, physics, and
component conventions. Treat snippets as patterns to adapt, not files to paste
unchanged.

## Start With Project Evidence

Before proposing nodes or code:

1. Locate the nearest `project.godot`. Read `config/features` and, when a Godot
   executable is available, confirm it with `godot --version` or the project's
   documented binary.
2. Inspect `application/run/main_scene`, `[input]`, named 2D physics layers,
   physics tick rate, default gravity, window/stretch settings, and texture
   filtering.
3. Search existing `.gd`, `.tscn`, and `.tres` files for player bodies,
   `TileMap` or `TileMapLayer`, animation nodes, damage APIs, groups, autoloads,
   and test tooling.
4. Preserve node names, typed-script style, signal naming, scene ownership, and
   collision conventions already in use.
5. Do not migrate a working `TileMap` merely to match an example. For new work,
   prefer `TileMapLayer` only when the inspected Godot version supports it.

If no project file exists, ask for the target Godot 4.x minor version and a
minimal scene contract before generating project-specific files.

## Choose the Physics Owner

- Use `CharacterBody2D` for code-driven actors that must slide against world
  collision. Use grounded motion for platformers and floating motion for
  top-down actors.
- Use `RigidBody2D` for simulation-driven objects. Apply impulses for discrete
  hits and forces for sustained acceleration; do not overwrite its transform
  each frame.
- Use `AnimatableBody2D` for scripted moving platforms and hazards that should
  transfer motion predictably.
- Use `Area2D` for overlap-only sensors, pickups, hurtboxes, and most simple
  projectiles. It does not provide swept body collision.
- Keep physics movement and queries in `_physics_process()`. Keep presentation
  updates in `_process()` unless they must match the physics result exactly.

## Implementation Workflow

1. **Define behavior in measurable terms.** Record speed, acceleration,
   stopping distance, jump height or airtime, attack cadence, invulnerability,
   and expected collision partners.
2. **Create inputs first.** Reuse semantic Input Map actions. Never hardcode
   keys in actor logic. Verify keyboard and controller dead zones where both
   are supported.
3. **Build a gray-box physics scene.** Use primitive collision shapes and a
   plain floor before adding art, animation, cameras, or combat.
4. **Implement one movement mode.** Keep velocity in pixels per second;
   `CharacterBody2D.move_and_slide()` applies the physics timestep internally.
5. **Build the world in layers.** Separate ground, collision-bearing terrain,
   decoration, foreground, and navigation according to actual responsibilities.
6. **Derive visuals from gameplay state.** Select animation after movement and
   avoid letting animation tracks overwrite authoritative physics transforms.
7. **Add combat as explicit interactions.** Give attacks, hurtboxes, health,
   teams, cooldowns, and death separate contracts. Use signals for feedback.
8. **Add game feel last.** Layer readable effects around confirmed events;
   never use shake or animation to hide unreliable collision.
9. **Verify at edges and under load.** Test slopes, corners, low ceilings,
   simultaneous hits, scene reloads, pooled reuse, and the highest expected
   actor/projectile count.

## Typed Platformer Core

This controller uses rate-based acceleration, buffered jumping, coyote time,
and a short-hop release. Tune values against the project's scale.

```gdscript
extends CharacterBody2D

@export_range(0.0, 2000.0, 1.0) var run_speed: float = 260.0
@export_range(0.0, 10000.0, 1.0) var ground_accel: float = 2200.0
@export_range(0.0, 10000.0, 1.0) var air_accel: float = 900.0
@export_range(0.0, 10000.0, 1.0) var gravity: float = 1800.0
@export_range(0.0, 5000.0, 1.0) var jump_speed: float = 560.0
@export_range(0.0, 5000.0, 1.0) var max_fall_speed: float = 1100.0
@export_range(0.0, 0.5, 0.01) var coyote_time: float = 0.10
@export_range(0.0, 0.5, 0.01) var jump_buffer_time: float = 0.12

var _coyote_left: float = 0.0
var _jump_buffer_left: float = 0.0

func _physics_process(delta: float) -> void:
    if Input.is_action_just_pressed(&"jump"):
        _jump_buffer_left = jump_buffer_time
    else:
        _jump_buffer_left = maxf(_jump_buffer_left - delta, 0.0)

    var axis: float = Input.get_axis(&"move_left", &"move_right")
    var accel: float = ground_accel if is_on_floor() else air_accel
    velocity.x = move_toward(velocity.x, axis * run_speed, accel * delta)

    if is_on_floor():
        _coyote_left = coyote_time
    else:
        _coyote_left = maxf(_coyote_left - delta, 0.0)
        velocity.y = minf(velocity.y + gravity * delta, max_fall_speed)

    if _jump_buffer_left > 0.0 and _coyote_left > 0.0:
        velocity.y = -jump_speed
        _jump_buffer_left = 0.0
        _coyote_left = 0.0

    if Input.is_action_just_released(&"jump") and velocity.y < 0.0:
        velocity.y *= 0.45

    move_and_slide()
```

Do not multiply `velocity` by `delta` before `move_and_slide()`. If jump height
must be exact, derive gravity and launch speed from the desired height and time
to apex instead of tuning both independently.

## Design Rules

- Size collision for playability, not sprite transparency. Keep feet stable and
  combat hurtboxes intentionally forgiving.
- Treat collision layers as an interface. Name each layer and document which
  masks consume it.
- Change animation only when the semantic state changes. Give airborne and
  action states priority over idle/run state.
- Apply damage once at the authoritative receiver. Feedback listeners may
  flash, shake, spawn particles, update UI, or play audio, but must not apply a
  second hit.
- Use timers or accumulated game delta for cooldowns. Wall-clock timestamps do
  not naturally honor pause or time scale.
- Pool only when profiling or scale justifies it. A pooled node needs explicit
  spawn/despawn reset for timers, monitoring, velocity, ownership, and process
  state; visibility alone is not a lifecycle.

## Verification Gate

Before finishing:

- Run the repository's existing test command and a headless parse/import check
  if a compatible Godot binary is available.
- Confirm every referenced Input Map action exists.
- Enable visible collision shapes and inspect tile collision, body shape, and
  Area2D masks in the running scene.
- Verify movement at the configured physics tick rate and after a window or
  camera scale change.
- Verify damage clamps, death emits once, friendly targets are ignored, and
  invulnerability cannot become permanent.
- For pools, watch the Remote scene tree and profiler: count should stabilize,
  and a reused object must behave like a fresh instance.
- Report files changed, scenes exercised, commands run, and any checks blocked
  by missing assets or a missing Godot executable.

## Common Failure Modes

- **Diagonal top-down movement is faster:** use `Input.get_vector()` or clamp
  vector length.
- **Character speed changes with frame rate:** keep movement in physics process
  and use `delta` only for rates, not the final CharacterBody velocity.
- **Floor state is always false:** check `up_direction`, motion mode, masks,
  collision shapes, and call order around `move_and_slide()`.
- **Tile collision looks offset:** compare TileSet tile size, atlas region size,
  layer transform, and collision polygons.
- **Animation flickers:** multiple branches call `play()` or state priority is
  ambiguous. Resolve one state, then apply it once.
- **An attack hits repeatedly:** disable or consume the hitbox, track victims per
  activation, or add receiver invulnerability.
- **Fast bullets tunnel:** use a swept body/ray query, shorter physics travel, or
  a shape cast rather than relying on overlap at the final position.
- **Shake leaves the camera displaced:** decay toward zero and explicitly reset
  offset when the effect ends.

## References

- [Movement, physics, and tilemaps](references/movement-physics-tilemaps.md)
- [Animation, combat, and game feel](references/animation-combat-game-feel.md)
- [Verification and troubleshooting](references/verification-and-troubleshooting.md)
