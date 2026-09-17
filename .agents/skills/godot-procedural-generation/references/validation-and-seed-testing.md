# Validation and Seed Testing

## Validate Independent Invariants

Return all useful failures instead of stopping at the first one during
development. Keep validators pure and deterministic.

```gdscript
const CARDINALS: Array[Vector2i] = [
    Vector2i.UP, Vector2i.RIGHT, Vector2i.DOWN, Vector2i.LEFT
]

func validate_cells(
    cells: Array[Vector2i], bounds: Rect2i, expected_count: int
) -> PackedStringArray:
    var errors: PackedStringArray = []
    if cells.size() != expected_count:
        errors.append("expected %d cells, got %d" % [expected_count, cells.size()])

    var occupied: Dictionary = {}
    for cell: Vector2i in cells:
        if not bounds.has_point(cell):
            errors.append("out of bounds: %s" % cell)
        if occupied.has(cell):
            errors.append("duplicate cell: %s" % cell)
        occupied[cell] = true

    if not cells.is_empty() and _reachable_count(cells[0], occupied) != occupied.size():
        errors.append("layout is disconnected")
    return errors

func _reachable_count(start: Vector2i, occupied: Dictionary) -> int:
    var seen: Dictionary = {start: true}
    var queue: Array[Vector2i] = [start]
    var cursor: int = 0
    while cursor < queue.size():
        var cell: Vector2i = queue[cursor]
        cursor += 1
        for direction: Vector2i in CARDINALS:
            var next: Vector2i = cell + direction
            if occupied.has(next) and not seen.has(next):
                seen[next] = true
                queue.append(next)
    return seen.size()
```

Add graph-specific validators for reciprocal edges, degree, loops, dead ends,
roles, and distances. Add content validators for room-resource compatibility,
spawn clearance, key/lock ordering, and encounter budgets.

## Property Tests Across Seeds

Use the repository's existing Godot test framework where present. Otherwise, a
headless test scene can run a deterministic batch and quit with a failing exit
code through the project's harness.

Core batch logic:

```gdscript
func verify_seed_range(
    generator: ConnectedRoomGenerator,
    bounds: Rect2i,
    room_count: int,
    seed_count: int
) -> void:
    for seed_value: int in range(seed_count):
        var first: Array[Vector2i] = generator.generate(
            seed_value, bounds, room_count
        )
        var second: Array[Vector2i] = generator.generate(
            seed_value, bounds, room_count
        )
        assert(not first.is_empty(), "failed seed %d" % seed_value)
        assert(validate_cells(first, bounds, room_count).is_empty())
        assert(cell_signature(first) == cell_signature(second))
```

Instantiate a new generator per run if it stores mutable search state. A result
for one seed must not depend on which seed ran before it.

## Test Classes

### Preconditions

- Zero or negative bounds.
- Requested count below minimum or above available cells.
- Empty room catalog.
- Duplicate room IDs.
- Missing start/exit definitions.
- Contradictory role quotas or distance requirements.

These should fail immediately without consuming the attempt budget.

### Boundary layouts

- One room.
- Exactly full bounds.
- Narrow one-cell-wide bounds.
- Nonzero and negative `Rect2i.position`.
- Largest supported room footprint and rotation.

### Determinism

- Same seed twice.
- Same seed before and after cosmetic RNG use.
- Seeds executed in ascending and descending order.
- Save/load manifest regeneration.
- Headless versus editor run on the same supported build.

### Structural properties

- Unique, in-bounds cells.
- Exact count or documented range.
- One connected component.
- Reciprocal edges and socket matches.
- Degree and loop/dead-end constraints.
- Role counts and graph-distance constraints.

### Failure behavior

- Impossible socket catalog.
- Search budget exhaustion.
- Missing PackedScene during realization.
- Validator rejection after a successful topology attempt.
- Regeneration failure while an old valid world is active.

Assert bounded completion and a useful error. Also assert that the active scene
or generated-content container did not change on failure.

## Distribution Checks

Determinism tests catch reproducibility errors, not poor variety. Across a large
seed sample, record:

- Template and role frequencies.
- Room count, dead ends, loops, and longest-path distribution.
- Attempt count and failure rate.
- Encounter, loot, and theme frequency.
- Generation duration and worst-case search work.

Use broad expected ranges rather than exact random frequencies. A statistically
unlikely batch should produce a diagnostic, while invariant violations should
fail immediately.

## Scene and Tile Validation

After realization and synchronization:

1. Count instantiated rooms and generated containers.
2. Compare each instance transform with canonical data.
3. Check door marker pairs and exterior seals.
4. Query spawn shapes for overlap with blocking collision.
5. Confirm every required tile cell has a valid source and atlas/terrain entry.
6. Query navigation between required points only after the map iteration is
   nonzero and the generation update has been observed.
7. Regenerate repeatedly and check that node and RID counts stabilize.

## Diagnosing a Failed Seed

Log a compact replay record:

```text
seed, schema, parameters, content version, attempt, stage, validator errors
```

Optionally retain the rejected canonical data in a debug-only result. Do not log
every random number by default; stage boundaries, selected stable IDs, and
constraint failures are usually enough to reproduce the issue.

When fixing a failed seed, add it to a regression set. Decide explicitly whether
the fix preserves old signatures or increments the generation schema.

## Completion Gate

- Project scripts parse with the inspected Godot version.
- Unit/property tests pass for representative and regression seeds.
- Impossible configurations terminate within the work budget.
- Same-seed canonical signatures match.
- Realization tests pass after server synchronization.
- The report includes tested seed range, worst attempt count, failures, commands
  run, changed files, and any unverified imported resources.
