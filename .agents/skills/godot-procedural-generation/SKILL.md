---
name: godot-procedural-generation
description: "Use when implementing or debugging deterministic procedural generation in Godot 4.x: seeded RNG, roguelike rooms, dungeon graphs, constrained placement, modular room sockets, TileMapLayer output, retries, connectivity validation, reproducible saves, or many-seed tests. Triggers on RandomNumberGenerator, room generation, level seeds, BSP, random walks, weighted content, generation failures, and validation."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Godot Procedural Generation

Build procedural systems as reproducible data transformations with explicit
constraints. Generate and validate a layout before mutating the scene tree.

## Inspect Before Designing

1. Locate `project.godot` and read its Godot feature/version tag. Confirm the
   executable version when available.
2. Inspect current world scenes, grid scale, `TileMap`/`TileMapLayer` usage,
   TileSet source IDs and terrain setup, room scenes, entrance markers,
   navigation, save data, autoloads, and test framework.
3. Find every existing random call. Determine whether gameplay already owns a
   seed or `RandomNumberGenerator` instance.
4. Identify persisted procedural data. Changing generation order or rules can
   invalidate old seeds even if the public seed value is unchanged.
5. Preserve the project's supported APIs. Prefer `TileMapLayer` for new output
   only when the inspected Godot 4.x version supports it.

If requirements such as room count, bounds, or connectivity are unknown, ask
for them. Do not invent silent constraints.

## Define the Generation Contract

Write the contract before the algorithm:

- Input seed and generation schema version.
- Grid shape, coordinate origin, bounds, and room/world scale.
- Exact or ranged output count.
- Required connectivity and permitted loops or dead ends.
- Room degree, socket, overlap, spacing, and orientation rules.
- Required roles such as start, exit, boss, reward, or key rooms.
- Graph-distance and ordering constraints between roles.
- Attempt/work budget and explicit failure result.
- Data that must be saved versus regenerated.

Separate hard constraints, which reject a layout, from soft preferences, which
only affect scoring or weights. An impossible hard-constraint set must fail in
bounded time rather than loop forever.

## Use a Staged Pipeline

```text
request -> seeded layout data -> structural validation -> content assignment
        -> semantic validation -> scene/tile realization -> server sync -> publish
```

- Keep layout cells, edges, sockets, room roles, and spawn records as data.
- Give generation a private RNG instead of using global `rand*` functions.
- Return a result carrying success, seed, data, attempt count, and failure
  reason. An empty level is not an adequate error channel.
- Commit nodes and tiles only after validation passes. Failed attempts should
  leave the active scene unchanged.
- Run presentation-only variation from a separate RNG stream so adding dust or
  decals cannot alter topology.

## Select an Algorithm by Constraint

- **Connected frontier growth:** reliable for a bounded room graph where every
  new room must touch the existing layout.
- **Random walk:** useful for winding footprints, but validate coverage and
  guard against repeated cells.
- **BSP partitioning:** useful for non-overlapping rectangular rooms and
  corridor construction.
- **Cellular automata:** useful for cave-like fields; retain the chosen connected
  component and repair or reject isolated regions.
- **Socket placement with backtracking:** useful for authored modular rooms.
  Filter compatible candidates first, then choose; cap search nodes.
- **AStarGrid2D or graph post-processing:** useful for corridor repair, lock/key
  ordering, or measuring path distance after placement.

Prefer an algorithm that satisfies the most important invariant by
construction. Validation remains mandatory because later stages can still
break it.

## Connected Seeded Layout

For a complete typed frontier-growth example, see [Deterministic pipelines and
RNG streams](references/deterministic-pipelines.md#connected-frontier-example).
It keeps search in data, uses a private RNG, bounds attempts, and checks
connectivity before scene realization. In production, return a typed result
that distinguishes precondition failure, attempt exhaustion, and validation
failure.

## Determinism Rules

- Set `rng.seed` once before consuming values. Do not call `randomize()` on a
  seeded generation RNG.
- Pass the RNG into helpers or keep it on one generator instance. Hidden random
  calls make reproduction fragile.
- Keep candidate ordering stable before random indexing. Do not depend on
  dictionary iteration order for random call order.
- Use separate seeded streams for topology, room roles, encounters, loot, and
  cosmetics when those stages must evolve independently.
- Save the seed, generation schema, parameters, relevant content version, and
  Godot version. A seed alone does not freeze changing algorithms or resources.
- Treat RNG algorithm details as engine internals. Reproducibility guarantees
  should name the supported build/version range.

## Validation Gate

Validate at least:

- Count, uniqueness, bounds, and no spatial overlap.
- Connectivity from the designated start.
- Reciprocal graph edges and compatible opposing sockets.
- Required role counts and minimum graph distances.
- Reachable doors, corridors, spawns, and objectives after content placement.
- Resource presence and valid `PackedScene` roots before instantiation.
- Tile source/terrain IDs against the actual TileSet.
- Navigation reachability only after navigation data has synchronized.

Run the same seed twice and compare canonical layout data. Then run hundreds or
thousands of seeds with a bounded timeout. Include impossible settings and
minimum/maximum bounds as tests.

## Common Failure Modes

- **Same seed, different layout:** global RNG use, unstable candidate order,
  async completion order, or an added random call in an earlier stage.
- **Generation hangs:** unbounded retry, recursive search without a work budget,
  or impossible constraints.
- **Fewer rooms than requested:** the frontier exhausted; return failure and
  retry or relax a documented soft constraint.
- **Rooms touch but doors disagree:** adjacency was inferred separately by each
  room. Build one graph edge and configure both ends from it.
- **Valid data becomes an invalid scene:** realization uses different units,
  origins, rotations, or room dimensions from validation.
- **Old generated nodes remain:** generation committed before validation or
  cleanup ownership is unclear.
- **Physics/navigation tests fail immediately after painting:** TileMap and
  server updates are batched; wait for synchronization.
- **A tool script corrupts authored content:** generated and authored nodes share
  a container, or editor execution lacks an explicit regeneration guard.

## References

- [Deterministic pipelines and RNG streams](references/deterministic-pipelines.md)
- [Room constraints and realization](references/room-constraints-and-realization.md)
- [Validation and seed testing](references/validation-and-seed-testing.md)
