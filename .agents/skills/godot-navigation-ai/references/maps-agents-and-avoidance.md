# Navigation Maps, Agents, and Avoidance

## Build the Traversable Surface

Use one of the project's established sources:

- A `NavigationRegion2D` containing an authored or baked
  `NavigationPolygon` for continuous arenas.
- TileSet navigation polygons exposed through a supported TileMap/TileMapLayer
  workflow for tile-authored worlds.
- Direct `NavigationServer2D` maps and regions only when node-based ownership is
  insufficient and the project already manages RIDs carefully.

Static walls should shape the navigation surface. A dynamic
`NavigationObstacle2D` can influence avoidance, but it does not automatically
turn every moving object into a pathfinding hole. If a building permanently
changes routes, update/rebake the relevant navigation data through the API
supported by the inspected Godot minor version.

Region edges connect only when their geometry and connection margin permit it.
Use `NavigationLink2D` for deliberate off-mesh transitions such as jumps,
teleports, doors, or one-way crossings. Put terrain capabilities on navigation
layers and ensure each agent's layer mask includes only legal regions/links.

## Respect Map Synchronization

Navigation node and server changes are generally applied during a later server
sync. Guard path work:

```gdscript
func navigation_map_is_ready(agent: NavigationAgent2D) -> bool:
    var map: RID = agent.get_navigation_map()
    return NavigationServer2D.map_get_iteration_id(map) > 0
```

For runtime map edits, remember the last observed iteration ID and wait until it
changes before validating new routes. A deferred call may be enough for a
simple startup, but an observed map update is a stronger contract than an
arbitrary frame count.

## Path-Following Invariants

- Set `target_position` in global coordinates.
- Check `is_navigation_finished()` before requesting the next point.
- Call `get_next_path_position()` exactly once per physics update while active;
  that call advances the agent's internal path state.
- Move the parent yourself. The agent is not a motor.
- Keep one movement authority. Animation, steering, knockback, and navigation
  must combine through a defined velocity policy rather than writing transforms
  independently.
- On teleport, move the body, reset its motion, and call the agent's forced
  velocity reset when the inspected API supports it.

### Typed Navigation Unit

```gdscript
class_name NavigationUnit2D
extends CharacterBody2D

signal navigation_finished(reached_target: bool)

@export_range(0.0, 2000.0, 1.0) var move_speed: float = 160.0
@onready var agent: NavigationAgent2D = %NavigationAgent2D

var _has_destination: bool = false
var _requested_target: Vector2 = Vector2.ZERO
var _target_submitted: bool = false

func _ready() -> void:
    motion_mode = CharacterBody2D.MOTION_MODE_FLOATING
    agent.max_speed = move_speed
    agent.velocity_computed.connect(_on_velocity_computed)

func issue_move(world_target: Vector2) -> void:
    _has_destination = true
    _requested_target = world_target
    _target_submitted = false

func stop_navigation() -> void:
    _has_destination = false
    _target_submitted = false
    agent.target_position = global_position
    agent.velocity = Vector2.ZERO
    velocity = Vector2.ZERO

func _physics_process(_delta: float) -> void:
    if not _has_destination:
        _submit_velocity(Vector2.ZERO)
        return

    var map: RID = agent.get_navigation_map()
    if NavigationServer2D.map_get_iteration_id(map) == 0:
        _submit_velocity(Vector2.ZERO)
        return

    if not _target_submitted:
        agent.target_position = _requested_target
        _target_submitted = true

    if agent.is_navigation_finished():
        var reached_target: bool = agent.is_target_reached()
        _has_destination = false
        _submit_velocity(Vector2.ZERO)
        navigation_finished.emit(reached_target)
        return

    var next_point: Vector2 = agent.get_next_path_position()
    var desired_velocity: Vector2 = global_position.direction_to(next_point) * move_speed
    _submit_velocity(desired_velocity)

func _submit_velocity(desired_velocity: Vector2) -> void:
    if agent.avoidance_enabled:
        agent.velocity = desired_velocity
    else:
        _apply_velocity(desired_velocity)

func _on_velocity_computed(safe_velocity: Vector2) -> void:
    _apply_velocity(safe_velocity if _has_destination else Vector2.ZERO)

func _apply_velocity(next_velocity: Vector2) -> void:
    velocity = next_velocity
    move_and_slide()
```

An unreachable target can still produce a path to a nearest reachable point.
Use both reachability/target state and the final position when command semantics
need to distinguish success from partial completion.

## Destination Projection

If the game clamps ground commands to the nearest walkable point, make that
behavior visible and bound the clamp distance:

```gdscript
func closest_legal_point(
    agent: NavigationAgent2D, requested: Vector2, max_snap: float
) -> Variant:
    var map: RID = agent.get_navigation_map()
    if NavigationServer2D.map_get_iteration_id(map) == 0:
        return null
    var closest: Vector2 = NavigationServer2D.map_get_closest_point(map, requested)
    if closest.distance_to(requested) > max_snap:
        return null
    return closest
```

Low-level closest-point queries use the map. If agents use different navigation
layers or maps, validate that the projected point belongs to terrain legal for
that unit before accepting it.

## Tune Distances From Motion

Start with the distance traveled per physics tick:

```text
step_distance = max_speed / physics_ticks_per_second
```

Set path-point tolerance comfortably above numerical noise and high enough that
the unit does not skip over it in one tick. Set target tolerance to the desired
arrival radius. Then test high-speed corners. `path_max_distance` should permit
normal avoidance deviation without allowing the unit to abandon the corridor.

The NavigationAgent `radius` does not widen a path. Bake navigation polygons
for the actor size or maintain separate maps for materially different sizes.

## Avoidance Integration

Enable avoidance, connect `velocity_computed`, and submit desired velocity each
physics tick. Apply only the returned safe velocity when avoidance is enabled.

```gdscript
func configure_avoidance(agent: NavigationAgent2D, speed: float, body_radius: float) -> void:
    agent.max_speed = speed
    agent.radius = body_radius
    agent.avoidance_enabled = true

func submit_desired_velocity(agent: NavigationAgent2D, desired: Vector2) -> void:
    agent.velocity = desired.limit_length(agent.max_speed)
```

Important tuning controls:

- `neighbor_distance`: search radius and a major cost multiplier.
- `max_neighbors`: maximum nearby agents considered.
- `time_horizon_agents`: how early agents react to other movers.
- `time_horizon_obstacles`: how early they react to avoidance obstacles.
- `avoidance_priority`: which agents yield more.
- `avoidance_layers` and `avoidance_mask`: crowd interaction groups.

Large neighbor ranges and horizons can make units slow preemptively. Small
values can produce late reactions. Test representative density rather than two
isolated units.

Avoidance has no knowledge of navigation polygons or body collision. A safe
RVO velocity can leave the mesh near an edge. Apply it through a physics body,
monitor path deviation, and design enough walkable clearance around bottlenecks.

## Static and Dynamic Obstacle Tests

1. Draw navigation debug geometry and confirm static walls are absent from the
   traversable surface.
2. Route around each obstacle from multiple directions.
3. Add and remove a runtime map-changing obstacle; wait for map iteration
   change before checking the new path.
4. Cross two avoidance-enabled units at an angle and head-on.
5. Send two groups through one chokepoint in opposing directions.
6. Test units with different avoidance layers and priorities.
7. Place a target outside the map and verify the intended reject/clamp/partial
   path behavior.

## Failure Diagnosis

- A path crossing a wall indicates wrong or stale navigation geometry, not an
  avoidance problem.
- A path that exists but cannot be followed indicates movement collision,
  narrow clearance, tolerance, or movement-authority problems.
- A constantly changing path usually means frequent target assignment or
  `path_max_distance` below ordinary steering deviation.
- Agents passing through each other usually lack matching avoidance masks,
  submit no velocity, or move without using the safe velocity callback.
- Agents stopping far apart usually have oversized avoidance radii, long time
  horizons, or one shared destination with no formation spacing.
