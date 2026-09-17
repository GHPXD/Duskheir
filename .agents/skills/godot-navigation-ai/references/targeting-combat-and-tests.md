# Targeting, Combat AI, and Tests

## Separate Intent From Execution

A useful state set is:

```text
IDLE -> MOVE
IDLE/MOVE -> CHASE -> ATTACK
CHASE/ATTACK -> IDLE when target is invalid or disengages
ANY -> DEAD
```

Commands or scanners choose intent. The unit state machine owns navigation,
range checks, cooldowns, animation requests, and transitions. Do not let a
scanner move the body directly.

Define target validity once:

```gdscript
func target_is_valid(target: CommandableUnit2D, own_team: int) -> bool:
    return (
        is_instance_valid(target)
        and not target.is_queued_for_deletion()
        and target.team_id != own_team
    )
```

Extend it with alive, visible, targetable, faction, or diplomacy rules from the
project. Recheck immediately before damage is committed.

## Acquire Targets at a Controlled Rate

Use a `Timer`, spatial registry, detection `Area2D`, or manager query. Groups are
adequate for modest counts when already used by the project:

```gdscript
func nearest_hostile(
    actor: CommandableUnit2D,
    group_name: StringName,
    acquire_range: float
) -> CommandableUnit2D:
    var best: CommandableUnit2D = null
    var best_distance_squared: float = acquire_range * acquire_range
    for node: Node in actor.get_tree().get_nodes_in_group(group_name):
        if not node is CommandableUnit2D:
            continue
        var candidate: CommandableUnit2D = node as CommandableUnit2D
        if not target_is_valid(candidate, actor.team_id):
            continue
        var distance_squared: float = actor.global_position.distance_squared_to(
            candidate.global_position
        )
        if distance_squared < best_distance_squared:
            best = candidate
            best_distance_squared = distance_squared
    return best
```

For large battles, maintain faction registries and spatial buckets instead of
allocating a full group list for every scan. Stagger scan timers so all units do
not query on the same frame.

Use an acquire radius smaller than the disengage radius. Keep the current valid
target unless a replacement is meaningfully better; otherwise two similar
targets can cause thrashing.

## Chase Without Repath Churn

Track the last requested target position and a repath interval. Request a new
path only when either threshold is reached:

```gdscript
@export_range(0.01, 2.0, 0.01) var chase_repath_interval: float = 0.20
@export_range(1.0, 256.0, 1.0) var chase_repath_distance: float = 24.0

var _repath_left: float = 0.0
var _last_target_position: Vector2 = Vector2.ZERO
var _has_last_target_position: bool = false

func update_chase(
    delta: float, unit: NavigationUnit2D, target_position: Vector2
) -> void:
    _repath_left = maxf(_repath_left - delta, 0.0)
    var target_moved: bool = (
        not _has_last_target_position
        or _last_target_position.distance_to(target_position) >= chase_repath_distance
    )
    if _repath_left > 0.0 and not target_moved:
        return
    _repath_left = chase_repath_interval
    _last_target_position = target_position
    _has_last_target_position = true
    unit.issue_move(target_position)
```

Stop navigation when combat range is reached. Resume chase only beyond a
slightly larger resume range to avoid toggling at the boundary. For ranged
attacks, require line of sight if walls should block fire.

## Line of Sight

Use a physics ray with an explicit world/target mask and exclude the attacker:

```gdscript
func has_line_of_sight(
    actor: CollisionObject2D,
    target: CollisionObject2D,
    collision_mask: int
) -> bool:
    var query: PhysicsRayQueryParameters2D = PhysicsRayQueryParameters2D.create(
        actor.global_position, target.global_position, collision_mask, [actor.get_rid()]
    )
    query.collide_with_areas = true
    query.collide_with_bodies = true
    var hit: Dictionary = actor.get_world_2d().direct_space_state.intersect_ray(query)
    return not hit.is_empty() and hit["collider"] == target
```

Call this helper during `_physics_process()` or another physics-safe boundary.
Queue line-of-sight requests raised by input, idle timers, or asynchronous work
for the next physics tick; direct-space state may be locked outside it.

Match the query to whether targets are bodies or areas. Decide whether allied
units block shots and encode that in masks/exclusions, not ad hoc name checks.

## Attack on Game Time

Accumulate physics delta or use a `Timer` configured for the desired pause/time
scale behavior:

```gdscript
@export_range(0.01, 10.0, 0.01) var attack_period: float = 0.75
var _attack_cooldown: float = 0.0

func tick_attack(delta: float, target: CommandableUnit2D) -> bool:
    _attack_cooldown = maxf(_attack_cooldown - delta, 0.0)
    if _attack_cooldown > 0.0:
        return false
    if not target_is_valid(target, team_id):
        return false
    _attack_cooldown = attack_period
    target.receive_damage(attack_damage, self)
    return true
```

The containing combat-unit script supplies `team_id`, `attack_damage`, and the
typed `receive_damage()` contract. For animation-timed melee, begin the attack
after cooldown but apply damage at one named animation event. Guard that event
with an attack token so interrupted or reused actors cannot deal a late hit.

## Target Cleanup

- Subscribe once to a target's death/tree-exit signal if the project exposes
  one, and disconnect when changing target.
- Also check `is_instance_valid()` because queued deletion and signal order can
  still produce gaps.
- Clear navigation intent when a target disappears.
- Remove dead units from faction registries and selection.
- Emit death before queueing deletion, but ensure listeners cannot deal another
  hit during teardown.

## AI and Combat Verification

### Deterministic state tests

- No target remains idle.
- Hostile inside acquire range becomes the target.
- Target between acquire and disengage ranges is retained but not newly
  acquired.
- Move command clears attack target.
- Attack command rejects allies and dead units.
- Chase repaths no faster than configured unless displacement threshold is met.
- Range hysteresis prevents chase/attack oscillation.
- Cooldown produces the expected hit count under pause and time scale.
- Target death returns the actor to the intended state without errors.

### Navigation scenarios

- Static target around each obstacle.
- Moving target that crosses navigation regions and links.
- Target outside or disconnected from the map.
- Knockback that pushes the unit off its ideal path.
- Teleport followed by a new command.
- Narrow doorway relative to baked clearance and body width.

### Crowd scenarios

- Many allies chasing one target.
- Two formations crossing at right angles.
- Opposing groups entering one chokepoint.
- Dormant units with avoidance disabled, then activated.
- Mixed priorities, radii, and avoidance masks.

Record frame time, navigation process counts, active avoidance agents, repath
rate, and target-scan rate at the expected unit count. Optimize only the measured
bottleneck.

## Failure Diagnosis

- AI attacks through walls: range has no line-of-sight rule or ray masks are
  wrong.
- AI changes targets constantly: no retention bias/hysteresis, or scans all
  candidates every frame.
- AI chases a freed object: target cleanup and validity checks are incomplete.
- Attack speed changes while paused: wall-clock time is being used or Timer
  process mode is wrong.
- Units orbit a target: stop/resume ranges, target tolerance, and avoidance
  radius conflict.
- Large fights spike periodically: scan timers align or every moving target is
  repathed on the same frame.

## Completion Report

Report the inspected Godot version, changed files, map source, unit root/body
type, navigation and avoidance settings, input paths tested, automated commands,
maximum units profiled, and any untested map updates or target-device behavior.
