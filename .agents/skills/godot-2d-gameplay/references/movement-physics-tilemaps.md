# Movement, Physics, and Tilemaps

## Establish Scale Before Tuning

Read the viewport size, camera zoom, asset dimensions, and TileSet tile size.
Values that feel responsive at 16-pixel tiles can be unusable at 128-pixel
tiles. Express movement goals in world units and seconds:

- Top speed in pixels per second.
- Time to reach top speed and time to stop.
- Platformer apex height and time to apex.
- Minimum corridor width relative to the body shape.

For a target jump height `h` and time to apex `t`, a useful starting point is:

```text
gravity = 2 * h / (t * t)
jump_speed = gravity * t
```

Godot's positive Y axis points down, so apply `-jump_speed` at takeoff.

## Top-Down CharacterBody2D

Set floating motion so every contact behaves as a wall. `Input.get_vector()`
applies configured dead zones and limits vector length to 1 while preserving
analog magnitude.

```gdscript
extends CharacterBody2D

@export_range(0.0, 2000.0, 1.0) var max_speed: float = 320.0
@export_range(0.0, 10000.0, 1.0) var acceleration: float = 2400.0
@export_range(0.0, 10000.0, 1.0) var deceleration: float = 3000.0

func _ready() -> void:
    motion_mode = CharacterBody2D.MOTION_MODE_FLOATING

func _physics_process(delta: float) -> void:
    var input_dir: Vector2 = Input.get_vector(
        &"move_left", &"move_right", &"move_up", &"move_down"
    )
    var target: Vector2 = input_dir * max_speed
    var rate: float = acceleration if not input_dir.is_zero_approx() else deceleration
    velocity = velocity.move_toward(target, rate * delta)
    move_and_slide()
```

If analog input magnitude should control speed, confirm that the project's
Input Map dead zones preserve that magnitude. If movement must remain on a
discrete grid, do not disguise grid steps as a free-moving CharacterBody; use a
turn/step state and reserve the next cell explicitly.

## Platformer Contact Details

- Set `up_direction` to the world's up vector; the default is `Vector2.UP`.
- Use `floor_snap_length` when actors should follow descending slopes. Do not
  force snap while rising from a jump.
- Tune `floor_max_angle` against authored slopes.
- Read floor and wall state from the previous `move_and_slide()` result when
  making decisions at the start of the next physics tick.
- Use `get_slide_collision_count()` when gameplay depends on what was hit. Do
  not infer a specific collider from `is_on_wall()` alone.
- Keep moving platforms as physics-aware bodies and test platform leave
  behavior. Teleporting a plain `Node2D` floor does not transfer velocity.

## RigidBody2D Boundaries

Use a rigid body when momentum, mass, torque, and contact response are part of
the mechanic. Trigger a launch once:

```gdscript
extends RigidBody2D

@export_range(0.0, 5000.0, 1.0) var impulse_strength: float = 650.0

func launch_toward(world_target: Vector2) -> void:
    var direction: Vector2 = global_position.direction_to(world_target)
    if not direction.is_zero_approx():
        apply_central_impulse(direction * impulse_strength)
```

Do not call an impulse continuously while an action is held. Use force from
`_integrate_forces()` or each physics tick for sustained thrust. Prefer physics
materials, damping, mass, and forces over direct transform writes.

## TileMapLayer Layout

When supported by the project version, use one `TileMapLayer` per functional
layer. A practical scene may contain:

- `Ground`: visual floor, usually no collision.
- `Terrain`: walls and slopes with TileSet physics polygons.
- `Details`: visual-only decals and props.
- `Foreground`: occluding artwork with deliberate draw order.
- `Navigation`: navigation data when the project derives it from tiles.

Share a saved `TileSet` resource where layers use the same atlas. Verify its
tile size, atlas region size, physics layers, terrain sets, custom data, and
navigation layers before painting or scripting cells.

TileMapLayer coordinates are local to the layer. Convert world positions
through the node transform before converting to a cell:

```gdscript
@export var terrain: TileMapLayer

func world_to_cell(world_position: Vector2) -> Vector2i:
    return terrain.local_to_map(terrain.to_local(world_position))

func cell_to_world(cell: Vector2i) -> Vector2:
    return terrain.to_global(terrain.map_to_local(cell))

func paint_cells(
    cells: Array[Vector2i], source_id: int, atlas_coords: Vector2i
) -> void:
    for cell: Vector2i in cells:
        terrain.set_cell(cell, source_id, atlas_coords)
```

Resolve `source_id`, atlas coordinates, alternative IDs, and terrain IDs from
the inspected TileSet. Never assume tutorial IDs. Prefer terrain connection
methods for autotiled boundaries when the TileSet is configured for them.

Tile updates are batched. Call `update_internals()` only when same-frame logic
truly requires rebuilt internals; frequent forced updates defeat batching. For
physics or navigation queries after runtime painting, allow the corresponding
server to synchronize before asserting results.

## Collision Matrix Review

For each collision object, write down:

```text
layer: what this object is
mask:  what this object needs to detect or block against
```

Check both physical blocking and overlap monitoring. An `Area2D` also needs
`monitoring`/`monitorable` in the correct direction. Keep world, actors,
projectiles, hitboxes, hurtboxes, pickups, and sensors on intentional named
layers rather than relying on the all-on default.

## Focused Tests

1. Hold each cardinal direction, then a diagonal; compare measured speed.
2. Accelerate and release input; verify stopping time at two physics tick rates.
3. Walk across seams, slopes, one-tile ledges, and moving platforms.
4. Jump on the first and last coyote/buffer tick and under a low ceiling.
5. Convert representative world points to cells and back; allow only the
   expected center offset.
6. Enable visible collision shapes and inspect every painted collision tile.
7. Paint and erase a runtime tile, wait for synchronization, then verify visual,
   collision, and navigation state separately.

## Failure Diagnosis

- Jitter at rest usually means competing movement writers, a changing target,
  an unstable floor, or a collision margin mismatched to world scale.
- Corner sticking often comes from a rectangular body, excessive safe margin,
  or level gaps narrower than the body.
- Blurry pixel art is usually texture filtering, fractional camera motion, or
  non-integer scaling, not a TileMap bug.
- A visually present tile without collision usually lacks a TileSet physics
  polygon or uses a physics layer the actor does not mask.
- A runtime cell at the wrong location usually skipped `to_local()` or applied
  the layer transform twice.
