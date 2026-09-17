---
name: godot-navigation-ai
description: "Godot 4.x pathfinding and movement-intent AI using NavigationRegion2D, NavigationAgent2D, avoidance, RTS unit selection, formations, target acquisition, and chase states. Use when requests concern navigation maps, unreachable targets, path jitter, local avoidance, unit commands, or crowd movement; defer damage and combat mechanics to godot-2d-gameplay."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Godot Navigation and AI

Treat pathfinding, movement, selection, commands, targeting, combat, and local
avoidance as separate systems with explicit handoffs.

## Inspect the Project First

1. Locate `project.godot`, read its Godot feature/version tag, and confirm the
   runtime version when a project-compatible executable is available.
2. Inspect navigation regions, polygons, tile navigation data, links, obstacles,
   maps, layers, and existing `NavigationAgent2D` settings.
3. Inspect unit roots, collision shapes, teams, groups/registries, selection UI,
   input actions, camera transforms, combat APIs, and pause behavior.
4. Search for direct position writes, `move_and_slide()`, target updates, physics
   queries, and navigation-server calls. Identify which script currently owns
   movement.
5. Preserve existing scene inheritance and command interfaces. Prefer
   `CharacterBody2D` plus `NavigationAgent2D` for new colliding units when the
   inspected project supports that architecture.

Do not replace a working `AStarGrid2D` grid with a navigation mesh without a
gameplay reason. Do not migrate APIs solely to match these examples.

## Keep the Models Distinct

- A navigation map describes traversable polygons and links.
- `NavigationAgent2D` requests and advances a path, but never moves its parent.
- `CharacterBody2D` owns collision-aware movement and final velocity.
- Static geometry belongs in baked or tile-provided navigation data.
- Avoidance modifies desired velocity around moving agents and avoidance
  obstacles; it does not change the path or understand physics collision.
- Selection identifies command recipients. Commands set intent; AI executes it.
- Combat range, line of sight, cooldown, and team rules are not pathfinding.

Use `AStarGrid2D` for strict cell movement or cheap grid costs. Use navigation
maps and agents for continuous movement through polygonal walkable space.

## Implementation Workflow

1. **Define locomotion constraints.** Record unit radius, speed, stopping
   distance, legal terrain, static/dynamic obstacles, and expected unit count.
2. **Build one valid map.** Author a `NavigationRegion2D` with a
   `NavigationPolygon`, or use inspected TileSet navigation data. Verify debug
   geometry before adding AI.
3. **Wait for synchronization.** Navigation-server changes usually become
   usable on a later physics update. Do not assume a path queried in `_ready()`
   is valid.
4. **Implement one unit and one static destination.** Call
   `get_next_path_position()` once per physics tick until navigation finishes.
5. **Add explicit commands.** A move order clears attack intent; an attack order
   owns a valid hostile target; stop/cancel clears both.
6. **Add selection behind UI.** Use `_unhandled_input()` or equivalent event
   routing so gameplay does not consume clicks handled by Controls.
7. **Add target/combat states.** Acquire on a timer or registry event, chase with
   throttled repaths, stop in range, attack on game-time cooldown, and recover
   safely when the target leaves or dies.
8. **Enable avoidance only where needed.** Connect `velocity_computed`, submit
   desired velocity, and apply the returned safe velocity through the body.
9. **Scale test.** Verify unreachable targets, narrow passages, opposing flows,
   formations, target death, pause, and the maximum expected crowd.

## Typed Navigation Unit

See the complete [typed navigation unit](references/maps-agents-and-avoidance.md#typed-navigation-unit)
for a pattern that guards map synchronization, avoids querying a finished path,
and keeps the physics body as the sole movement authority.

Set `path_desired_distance` and `target_desired_distance` from speed, physics
tick, body radius, and desired stop behavior. Values that are too small cause
overshoot/backtracking; values that are too large skip corners.

## Command and AI Rules

- Commands use global positions. Project ground clicks to a legal destination
  when the design allows clamping; otherwise reject unreachable orders visibly.
- Update a moving target's path only after it moved a threshold or a repath
  interval elapsed. Assigning `target_position` every frame can produce path
  churn and direction flicker.
- Keep selected units in a typed collection and remove invalid/freed entries.
- Give multi-unit moves distinct formation slots instead of one identical point.
- Validate team and target liveness both when accepting an attack order and
  immediately before dealing damage.
- Use game delta or `Timer` for attack cadence and scans. System wall time does
  not naturally honor pause and time scale.
- Use acquire/disengage hysteresis so targets do not toggle at one exact radius.
- Scan on a timer or maintain a registry; do not search the entire scene for
  every unit every frame.

## Avoidance Rules

- `avoidance_enabled` has a cost. Enable it for active movers in crowds, not
  automatically for every dormant unit.
- Set agent `max_speed` to at least the submitted desired speed.
- `radius` describes avoidance size only. Path clearance comes from navigation
  geometry and its bake settings; different body sizes may need separate maps.
- Tune `neighbor_distance`, `max_neighbors`, and time horizons downward from
  defaults while preserving required behavior.
- Avoidance layers/masks are independent of physics and navigation layers.
- Feed the safe velocity back through `CharacterBody2D.move_and_slide()` so
  physics remains a final barrier.
- RVO can move a unit away from the path or even outside navigable polygons.
  Keep `path_max_distance` sensible and test map boundaries.

## Verification Gate

- Show navigation debug geometry and each test agent's path.
- Assert the agent map iteration ID is nonzero before evaluating a path.
- Test a reachable point, an unreachable point, a point outside the map, and a
  destination across every intended link/layer boundary.
- Distinguish `is_navigation_finished()` from `is_target_reached()`; an
  unreachable path can finish at its closest reachable endpoint.
- Test selection and drag selection under camera pan/zoom and over UI.
- Test move, attack, cancel, additive selection, target death, friendly clicks,
  and commands issued while units are moving.
- Test avoidance with crossing streams and dense formations, then profile at the
  target unit count.
- Run existing tests and a headless parse/import check with the inspected Godot
  version when available.
- Report files changed, maps/scenes exercised, commands run, unit counts tested,
  and checks blocked by missing assets or executable.

## Common Failure Modes

- **Empty path at startup:** map has not synchronized. Queue the command until a
  map update is observed or the iteration ID is nonzero.
- **Unit never moves:** the agent only computes path data; parent movement code
  is missing, or navigation layers/maps do not match.
- **Unit dances or looks backward:** target is reassigned too often, distances
  are below per-tick travel, or the agent is repeatedly forced off-path.
- **Unit cuts corners or exits the map:** path distance is too large, avoidance
  dominates steering, or physics/navigation geometry disagree.
- **Avoidance returns zero:** avoidance is disabled, no map is assigned, desired
  velocity/target was not supplied, or layers do not match.
- **Large units clip walls:** avoidance radius was mistaken for pathfinding
  clearance; rebake or use an appropriate map.
- **Selection clicks through UI:** input is handled too early or UI mouse filters
  are wrong.
- **Dead target crashes combat:** references are not checked with
  `is_instance_valid()` or cleared on target exit/death.
- **Crowds stall:** all units share one destination, horizons/radii are too
  conservative, or the map has a bottleneck narrower than the formation.

## References

- [Navigation maps, agents, and avoidance](references/maps-agents-and-avoidance.md)
- [Unit selection and commands](references/selection-and-commands.md)
- [Targeting, combat AI, and tests](references/targeting-combat-and-tests.md)
