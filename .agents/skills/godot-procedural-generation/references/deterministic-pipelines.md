# Deterministic Pipelines and RNG Streams

## Reproducibility Contract

State what "same seed" promises. A practical contract is:

```text
same seed + same generation schema + same parameters + same content catalog
+ same supported Godot build = same canonical generation data
```

Do not promise identical nodes across engine upgrades or content edits unless
the project snapshots generated data instead of regenerating it.

Persist a small manifest alongside a run or save:

```gdscript
func make_generation_manifest(
    seed_value: int, schema: int, parameters: Dictionary
) -> Dictionary:
    return {
        "seed": seed_value,
        "schema": schema,
        "parameters": parameters.duplicate(true),
        "godot": Engine.get_version_info(),
    }
```

Add a content-catalog version or hash when room resources, item tables, or
enemy definitions influence output.

## Own the RNG

Global random functions share state with every caller. Procedural systems
should own `RandomNumberGenerator` instances and expose deterministic methods.

```gdscript
class_name GenerationRandom
extends RefCounted

const TOPOLOGY_STREAM: int = 0x13579BDF
const CONTENT_STREAM: int = 0x2468ACE0
const COSMETIC_STREAM: int = 0x10203040

static func create(base_seed: int, stream_salt: int) -> RandomNumberGenerator:
    var rng: RandomNumberGenerator = RandomNumberGenerator.new()
    rng.seed = base_seed ^ stream_salt
    return rng
```

Fixed stream salts isolate stages. This is not cryptography; it is a way to
prevent a new cosmetic roll from shifting room topology. If externally supplied
seeds have poor patterns, normalize them through the project's documented hash
strategy and test neighboring seeds for unwanted correlation.

Use `rng.state` only to resume a previously captured point in the same stream.
Do not invent arbitrary state values. Set the seed before restoring state.

## Keep Random Call Order Stable

Determinism depends on both values and call order:

- Sort catalog entries by stable ID before weighted selection.
- Traverse cardinal directions in one declared order.
- Convert dictionary keys to a sorted array before iteration affects RNG calls.
- Avoid using node child order unless it is an explicit content contract.
- Do not let thread completion or signals decide which stage consumes the next
  random value.
- Keep validation pure; validators should not consume the generation RNG.
- Keep retries deterministic by using a documented stream progression or a
  deterministic attempt seed.

## Connected Frontier Example

This frontier-growth example rejects crowded placements, retries with one
private RNG stream, and never instantiates a room during search:

```gdscript
class_name ConnectedRoomGenerator
extends RefCounted

const DIRECTIONS: Array[Vector2i] = [
    Vector2i.UP, Vector2i.RIGHT, Vector2i.DOWN, Vector2i.LEFT
]

func generate(
    seed_value: int,
    bounds: Rect2i,
    target_count: int,
    max_attempts: int = 32
) -> Array[Vector2i]:
    if not bounds.has_area() or max_attempts <= 0:
        return []
    if target_count <= 0 or target_count > bounds.get_area():
        return []

    var rng: RandomNumberGenerator = RandomNumberGenerator.new()
    rng.seed = seed_value

    for _attempt: int in range(max_attempts):
        var cells: Array[Vector2i] = _attempt_layout(rng, bounds, target_count)
        if cells.size() == target_count and _is_connected(cells):
            return cells
    return []

func _attempt_layout(
    rng: RandomNumberGenerator, bounds: Rect2i, target_count: int
) -> Array[Vector2i]:
    var start := Vector2i(
        bounds.position.x + bounds.size.x / 2,
        bounds.position.y + bounds.size.y / 2
    )
    var cells: Array[Vector2i] = [start]
    var occupied: Dictionary = {start: true}
    var queued: Dictionary = {}
    var frontier: Array[Vector2i] = []
    _append_frontier(start, bounds, occupied, queued, frontier)

    while cells.size() < target_count and not frontier.is_empty():
        var index: int = rng.randi_range(0, frontier.size() - 1)
        var candidate: Vector2i = frontier[index]
        frontier.remove_at(index)
        queued.erase(candidate)

        if occupied.has(candidate):
            continue
        var neighbors: int = _occupied_neighbor_count(candidate, occupied)
        if neighbors == 0 or neighbors > 2:
            continue

        occupied[candidate] = true
        cells.append(candidate)
        _append_frontier(candidate, bounds, occupied, queued, frontier)

    return cells

func _append_frontier(
    origin: Vector2i,
    bounds: Rect2i,
    occupied: Dictionary,
    queued: Dictionary,
    frontier: Array[Vector2i]
) -> void:
    for direction: Vector2i in DIRECTIONS:
        var candidate: Vector2i = origin + direction
        if not bounds.has_point(candidate):
            continue
        if occupied.has(candidate) or queued.has(candidate):
            continue
        queued[candidate] = true
        frontier.append(candidate)

func _occupied_neighbor_count(cell: Vector2i, occupied: Dictionary) -> int:
    var count: int = 0
    for direction: Vector2i in DIRECTIONS:
        count += int(occupied.has(cell + direction))
    return count

func _is_connected(cells: Array[Vector2i]) -> bool:
    if cells.is_empty():
        return false
    var occupied: Dictionary = {}
    for cell: Vector2i in cells:
        occupied[cell] = true
    var reached: Dictionary = {cells[0]: true}
    var queue: Array[Vector2i] = [cells[0]]
    var cursor: int = 0
    while cursor < queue.size():
        var current: Vector2i = queue[cursor]
        cursor += 1
        for direction: Vector2i in DIRECTIONS:
            var neighbor: Vector2i = current + direction
            if occupied.has(neighbor) and not reached.has(neighbor):
                reached[neighbor] = true
                queue.append(neighbor)
    return reached.size() == cells.size()
```

Adjust degree rules only from the written generation contract. A production
result should explain why generation failed rather than returning only an empty
array.

## Separate Data From Nodes

Model a generated room without instantiating it:

```gdscript
class_name GeneratedRoom
extends RefCounted

var cell: Vector2i
var template_id: StringName
var role: StringName
var rotation_steps: int
var neighbors: Array[Vector2i] = []

func _init(at: Vector2i) -> void:
    cell = at
```

Store stable IDs in generation data, not scene-node references. Resolve IDs to
resources during realization. This makes layouts testable in headless mode and
keeps failed attempts out of the scene tree.

A robust result object distinguishes failures:

```gdscript
class_name GenerationResult
extends RefCounted

var ok: bool = false
var seed_value: int = 0
var attempts: int = 0
var rooms: Array[GeneratedRoom] = []
var error: String = ""
```

Do not publish partially populated `rooms` when `ok` is false unless the result
is explicitly a diagnostic artifact.

## Canonical Signatures

Compare canonical data, not instance IDs or child names. Sort first:

```gdscript
func cell_signature(cells: Array[Vector2i]) -> String:
    var ordered: Array[Vector2i] = []
    for cell: Vector2i in cells:
        ordered.append(cell)
    ordered.sort_custom(func(a: Vector2i, b: Vector2i) -> bool:
        return a.y < b.y or (a.y == b.y and a.x < b.x)
    )

    var parts: PackedStringArray = []
    for cell: Vector2i in ordered:
        parts.append("%d:%d" % [cell.x, cell.y])
    return ",".join(parts)
```

Extend the signature with sorted edges, template IDs, roles, and rotations when
those stages are part of the reproducibility contract.

## Pipeline Ownership

One coordinator should own the transaction:

1. Validate request preconditions without randomness.
2. Create stage RNGs.
3. Build topology data.
4. Validate topology.
5. Assign room/content data.
6. Validate semantic rules.
7. Realize into a fresh generated-content container.
8. Wait for required TileMap, physics, and navigation synchronization.
9. Publish the new container and dispose of the previous generated container.

For seamless regeneration, build the replacement off to the side, then swap it
in after success. Never clear the current playable world before knowing the new
one is valid.

## Determinism Tests

- Generate one seed twice in one process and compare signatures.
- Generate it again after unrelated visual RNG consumption.
- Generate seeds in different test order and compare per-seed signatures.
- Save and restore an RNG state and confirm the next sequence repeats.
- Verify validators and debug drawing do not alter later output.
- Run with the same project build in headless and editor modes.
- If cross-version reproduction is required, keep golden manifests and test
  every supported Godot version explicitly.
