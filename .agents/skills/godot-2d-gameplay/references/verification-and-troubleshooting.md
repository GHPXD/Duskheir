# Verification and Troubleshooting

## Run the Project's Checks First

Use the repository's documented commands and test framework. If no check exists
and a compatible Godot executable is installed, a useful parse/import smoke
check is:

```sh
godot --headless --path . --editor --quit
```

Use the actual project binary name and version. Do not import a Godot 4 project
with a different minor version merely to run a check. For a focused test scene,
use the project's normal runner or a scene that quits itself after assertions.

## Minimal Runtime Assertions

Place temporary assertions in an existing test harness, not production startup
code:

```gdscript
func assert_required_actions() -> void:
    var actions: Array[StringName] = [
        &"move_left", &"move_right", &"move_up", &"move_down", &"jump"
    ]
    for action: StringName in actions:
        assert(InputMap.has_action(action), "Missing Input Map action: %s" % action)
```

Adapt the list to the movement mode; a top-down game may not have `jump`.

Health invariants can be tested without rendering:

```gdscript
func assert_health_contract(health: HealthComponent) -> void:
    var deaths: Array[int] = [0]
    health.died.connect(func() -> void: deaths[0] += 1)
    assert(not health.apply_damage(0))
    assert(health.apply_damage(health.maximum + 50))
    assert(health.current == 0)
    assert(deaths[0] == 1)
    assert(not health.apply_damage(1))
    assert(deaths[0] == 1)
```

If the repository uses GUT, gdUnit4, or a custom harness, express these as its
native tests instead.

## Movement Test Scene

Build one small scene containing:

- Flat floor and a one-tile ledge.
- Inside and outside corners.
- The steepest supported slope and one unsupported slope.
- A low ceiling over a jump.
- A narrow passage at the minimum legal width.
- A moving platform if the game uses them.
- One wall and floor painted from the production TileSet.

Measure rather than eyeball:

- Time to top speed and time to stop.
- Distance traveled in one second horizontally and diagonally.
- Jump apex height and airtime.
- Coyote and buffer windows at their boundary ticks.
- Maximum penetration or visual separation at corners.

Repeat at the project's intended physics tick rate. Rendering FPS should not
alter physics outcomes materially.

## Tilemap Checks

1. Confirm the runtime node class matches the project version.
2. Inspect every TileMapLayer transform; gameplay layers should not have
   accidental scale or offset.
3. Confirm TileSet tile size and every atlas source region size.
4. Enable visible collision shapes and navigation debug drawing separately.
5. Check the actor mask against the TileSet physics layer.
6. Round-trip cells through `map_to_local()` and `local_to_map()`.
7. For generated cells, assert source IDs and atlas coordinates exist before
   painting.
8. After runtime edits, wait for batched TileMap, physics, and navigation updates
   before querying them.

## Combat Checks

Create a matrix of source and receiver teams and verify allowed interactions.
Then exercise:

- Spawn already overlapping a hurtbox.
- Enter, stay, and exit during one attack activation.
- Multiple hurtboxes on one actor.
- Two attackers in the same physics tick.
- Receiver death during a signal callback.
- Scene pause during cooldown or invulnerability.
- Projectile collision at maximum speed and minimum target thickness.
- Pool exhaustion and repeated reuse.

The Remote scene tree reveals duplicate managers, orphan projectiles, and pools
that continue growing. The profiler reveals whether pooling helps frame time or
only adds complexity.

## Symptom-Driven Diagnosis

### Character does not move

- Confirm the scene with the script is instantiated and processing.
- Confirm action names exactly match Input Map entries.
- Inspect whether another script overwrites `velocity` later in the tick.
- Confirm the body is not embedded in collision at startup.
- Check process mode when the tree is paused.

### Character passes through terrain

- Confirm a `CollisionShape2D` is enabled on the actor.
- Confirm the TileSet tile has a physics polygon.
- Compare actor mask with terrain layer.
- Ensure movement uses a physics body API rather than direct transform changes.
- For high speed, inspect travel per physics tick and use swept collision.

### Character sticks or jitters

- Look for two movement authorities: animation, tween, parent transform, and
  physics script are common conflicts.
- Check body shape, safe margin, floor snap, and corridor width at project scale.
- Remove visual code and reproduce in the gray-box scene.
- Log state transitions, not every frame, to find oscillation.

### Overlap signal never fires

- Check layer/mask direction, `monitoring`, and `monitorable`.
- Connect the correct signal: body versus area.
- Confirm the shape is enabled and the node is inside the active tree.
- Avoid enabling/disabling collision directly during a physics callback; defer
  the property change when required.

### Animation restarts or never completes

- Call `play()` only when the chosen clip changes.
- Check whether an `AnimationPlayer` property track fights script writes.
- Give one system ownership of action completion.
- Verify loop settings and animation names against the actual resource.

### Pooled object behaves inconsistently

- Replace visibility-based reuse with explicit active/free tracking.
- Stop and restart timers and particles.
- Clear signal subscriptions made per spawn.
- Reset collision, process mode, velocity, target, team, and victim history.
- Guard late callbacks with an activation token or `active` check.

## Completion Report

Report:

- Exact files changed.
- Godot version inferred and binary version used.
- Scenes and inputs inspected.
- Automated commands and manual scenarios run.
- Known untested paths, especially controller hardware, imported assets,
  navigation synchronization, or performance on target devices.
