# Room Constraints and Realization

## Author Room Metadata, Not Filename Rules

Represent room compatibility in a resource. Stable IDs are safer than deriving
meaning from scene paths or node names.

```gdscript
class_name RoomDefinition
extends Resource

enum Socket { NORTH = 1, EAST = 2, SOUTH = 4, WEST = 8 }

@export var id: StringName
@export var scene: PackedScene
@export_flags("North", "East", "South", "West") var sockets: int = 0
@export var tags: Array[StringName] = []
@export_range(0.0, 1000.0, 0.01) var weight: float = 1.0
@export_range(1, 64, 1) var width_cells: int = 1
@export_range(1, 64, 1) var height_cells: int = 1
```

For rotatable rooms, either author metadata per allowed orientation or rotate
both footprint and sockets through one tested function. Never rotate only the
visual scene.

Each room scene should expose deliberate markers or a typed setup method for
doors, spawns, bounds, and content anchors. Avoid deep hardcoded paths from the
generator into a room's private hierarchy.

## Graph and Socket Rules

Build adjacency once. A cardinal edge is reciprocal by definition:

```gdscript
const OPPOSITE: Dictionary = {
    Vector2i.UP: Vector2i.DOWN,
    Vector2i.RIGHT: Vector2i.LEFT,
    Vector2i.DOWN: Vector2i.UP,
    Vector2i.LEFT: Vector2i.RIGHT,
}

func has_reciprocal_edge(
    graph: Dictionary, from: Vector2i, direction: Vector2i
) -> bool:
    var to: Vector2i = from + direction
    if not graph.has(from) or not graph.has(to):
        return false
    var from_edges: Array = graph[from]
    var to_edges: Array = graph[to]
    return direction in from_edges and OPPOSITE[direction] in to_edges
```

Validate all edges from the graph. Then choose only room definitions whose
sockets cover the required edge directions and reject unwanted openings when
the design requires sealed exterior walls.

Useful hard constraints include:

- Footprint remains inside bounds and does not overlap occupied cells.
- Every graph edge has matching sockets on both rooms.
- Start and exit appear exactly once.
- Boss and reward rooms satisfy minimum distance from start.
- A locked edge is not the only route to its own key.
- Spawn markers lie on walkable ground and outside door clearance.

Useful soft preferences include:

- Fewer repeated templates in adjacent rooms.
- Desired dead-end or loop count.
- Encounter intensity that rises with graph distance.
- Visual themes clustered or alternated by region.

Do not reject an otherwise valid level for a soft preference; score candidates
or adjust weights instead.

## Weighted Choice After Filtering

Filter candidates by hard constraints before rolling a weight. This helper is
compatible with Godot 4.x versions that do not expose newer weighted helpers:

```gdscript
func choose_weighted(
    candidates: Array[RoomDefinition], rng: RandomNumberGenerator
) -> RoomDefinition:
    var total: float = 0.0
    for candidate: RoomDefinition in candidates:
        total += maxf(candidate.weight, 0.0)
    if total <= 0.0:
        return null

    var roll: float = rng.randf_range(0.0, total)
    var fallback: RoomDefinition = null
    for candidate: RoomDefinition in candidates:
        var weight: float = maxf(candidate.weight, 0.0)
        if weight <= 0.0:
            continue
        fallback = candidate
        roll -= weight
        if roll < 0.0:
            return candidate
    return fallback
```

Sort `candidates` by stable room ID before this call. Reject duplicate IDs when
loading the catalog.

## Bounded Backtracking

For modular rooms with many constraints:

1. Choose the unassigned location with the fewest valid candidates.
2. Build and stably sort its candidate list.
3. Use seeded order or weighted choice.
4. Place one candidate and propagate socket/footprint constraints.
5. Roll back all data changes if a later location has no candidate.
6. Stop at a configured search-node or time budget.
7. Return a diagnostic naming the exhausted location and constraint.

Track rollback data explicitly. Instantiating and freeing scenes during search
is slower and makes rollback error-prone.

## Realize a Valid Layout

For room scenes:

```gdscript
func instantiate_room(
    parent: Node2D,
    definition: RoomDefinition,
    cell: Vector2i,
    room_size: Vector2
) -> Node2D:
    if definition.scene == null:
        return null
    var instance: Node = definition.scene.instantiate()
    if not instance is Node2D:
        instance.free()
        return null
    var room: Node2D = instance as Node2D
    room.position = Vector2(cell) * room_size
    parent.add_child(room)
    return room
```

Configure a room through its typed public method after it enters the tree if it
depends on `@onready` children. For editor tools, set `owner` only when generated
nodes are intentionally serialized.

For tile output, use inspected TileSet IDs and separate functional layers:

```gdscript
func paint_floor(
    layer: TileMapLayer,
    cells: Array[Vector2i],
    source_id: int,
    atlas_coords: Vector2i
) -> void:
    for cell: Vector2i in cells:
        layer.set_cell(cell, source_id, atlas_coords)
```

Use `set_cells_terrain_connect()` when a configured terrain set should resolve
borders. Validate terrain IDs before calling it. Do not force
`update_internals()` after every cell; paint in batches.

## Synchronization Boundary

Realization can affect multiple servers on different schedules:

- TileMapLayer batches internal updates.
- Physics shapes become queryable on a physics update.
- Navigation map changes generally synchronize on a later physics frame.
- Deferred scene-tree changes are not present immediately.

Wait at a single documented boundary before spawning actors or running
reachability checks. For navigation, prefer a map-change signal or inspect the
navigation map iteration ID rather than assuming one arbitrary frame is always
enough.

## Realization Tests

- Every data room resolves to exactly one known resource.
- Every instantiated root has the expected type and setup API.
- World positions equal the declared grid transform, including nonzero origins.
- Connected doors align in world space within a small tolerance.
- Exterior sockets are sealed when required.
- Spawn markers are not inside collision after server synchronization.
- Regeneration replaces only generated content and leaves authored nodes intact.
