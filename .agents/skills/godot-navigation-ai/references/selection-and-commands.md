# Unit Selection and Commands

## Expose a Unit Command Interface

The controller should not reach into a unit's child nodes. Give each unit a
small public API:

```gdscript
class_name CommandableUnit2D
extends NavigationUnit2D

@export var team_id: int = 0
@export var selection_priority: int = 0
@onready var selection_visual: CanvasItem = %SelectionVisual

func set_selected(value: bool) -> void:
    selection_visual.visible = value

func issue_attack(target: CommandableUnit2D) -> void:
    if target == null or target.team_id == team_id:
        return
    # Hand off to the unit's combat-intent implementation.
```

Add stop, queued move, patrol, or interact commands only when required by the
game. Keep move and attack mutually clear unless the design explicitly supports
attack-move.

## Route Input Behind UI

Use `_unhandled_input()` so a `Control` can consume clicks first. Verify UI
`mouse_filter` settings. Input events provide viewport coordinates; convert an
accepted click to a world position and queue it. Run the physics query during
the next `_physics_process()` tick because direct-space access can be locked
outside physics processing, especially with threaded physics.

```gdscript
const MAX_SELECTION_HITS: int = 64
var _pending_clicks: Array[Dictionary] = []

func _unhandled_input(event: InputEvent) -> void:
    var mouse: InputEventMouseButton = event as InputEventMouseButton
    if mouse == null or not mouse.pressed:
        return
    if mouse.button_index != MOUSE_BUTTON_LEFT:
        return
    var world_position: Vector2 = (
        get_viewport().get_canvas_transform().affine_inverse() * mouse.position
    )
    _pending_clicks.append({
        "world_position": world_position,
        "additive": mouse.shift_pressed,
    })

func _physics_process(_delta: float) -> void:
    var requests: Array[Dictionary] = _pending_clicks
    _pending_clicks = []
    for request: Dictionary in requests:
        var clicked_unit: CommandableUnit2D = unit_at(request["world_position"])
        # Apply selection using clicked_unit and request["additive"].
```

For CharacterBody2D units, use a point query with an explicit unit mask:

```gdscript
@export_flags_2d_physics var selectable_mask: int

func unit_at(world_position: Vector2) -> CommandableUnit2D:
    var query: PhysicsPointQueryParameters2D = PhysicsPointQueryParameters2D.new()
    query.position = world_position
    query.collision_mask = selectable_mask
    query.collide_with_bodies = true
    query.collide_with_areas = false

    var hits: Array[Dictionary] = get_world_2d().direct_space_state.intersect_point(
        query, MAX_SELECTION_HITS
    )
    var matches: Array[CommandableUnit2D] = []
    var seen: Dictionary = {}
    for hit: Dictionary in hits:
        var collider: Object = hit["collider"]
        if collider is CommandableUnit2D:
            var instance_id: int = collider.get_instance_id()
            if seen.has(instance_id):
                continue
            seen[instance_id] = true
            matches.append(collider as CommandableUnit2D)
    if hits.size() == MAX_SELECTION_HITS:
        push_warning("Selection query reached MAX_SELECTION_HITS; priority may be incomplete.")
    if matches.is_empty():
        return null
    matches.sort_custom(_unit_has_higher_priority)
    return matches[0]

func _unit_has_higher_priority(
    left: CommandableUnit2D, right: CommandableUnit2D
) -> bool:
    if left.selection_priority != right.selection_priority:
        return left.selection_priority > right.selection_priority
    return String(left.get_path()) < String(right.get_path())
```

If units expose a separate clickable `Area2D`, invert the body/area flags and
resolve the owning unit through a typed method. Always define stable priority
for overlapping matches; physics-query result order is not guaranteed. Set the
query cap above the maximum legal overlap and treat cap saturation as a scene
or selection-contract error.

## Maintain Selection Safely

Store `Array[CommandableUnit2D]`, not node paths. Before each command, remove
entries that are no longer instance-valid or are queued for deletion. On unit
death/tree exit, remove it from selection and hide/clear group UI.

Selection semantics should be explicit:

- Plain click replaces selection.
- Modifier-click toggles or adds, if supported.
- Empty click clears unless a modifier preserves selection.
- Drag selects only selectable, alive, player-controlled units.
- Double-click/type selection is a separate feature, not an accidental timing
  branch in basic click handling.

For drag selection, keep the drag rectangle in viewport coordinates. Transform
each unit's global position through the viewport canvas transform so camera pan
and zoom are handled consistently:

```gdscript
func units_in_screen_rect(screen_rect: Rect2) -> Array[CommandableUnit2D]:
    var selected: Array[CommandableUnit2D] = []
    var canvas: Transform2D = get_viewport().get_canvas_transform()
    for node: Node in get_tree().get_nodes_in_group(&"selectable_units"):
        if not node is CommandableUnit2D:
            continue
        var unit: CommandableUnit2D = node as CommandableUnit2D
        var screen_position: Vector2 = canvas * unit.global_position
        if screen_rect.has_point(screen_position):
            selected.append(unit)
    return selected
```

If the project uses nested `SubViewport`s or CanvasLayers, derive coordinates
from the actual gameplay viewport rather than the root viewport.

## Decide Attack Versus Move

On a command click:

1. Clean the selection.
2. Query a commandable target under the cursor.
3. If it is hostile and attackable, issue attack intent.
4. If it is friendly, apply the project's friendly-click behavior.
5. Otherwise resolve a legal ground position and issue move intent.
6. Show command feedback at the accepted target, not merely the raw cursor.

Do not issue a move behind an enemy because the target query used the wrong
physics mask. Keep selectable/targetable masks explicit.

## Formation Destinations

Giving every selected unit one target creates an artificial avoidance jam.
Generate deterministic slots around the command point and assign them in a
stable unit order.

```gdscript
func grid_slots(center: Vector2, count: int, spacing: float) -> Array[Vector2]:
    var slots: Array[Vector2] = []
    if count <= 0:
        return slots
    var columns: int = ceili(sqrt(float(count)))
    var rows: int = ceili(float(count) / float(columns))
    var origin := center - Vector2(columns - 1, rows - 1) * spacing * 0.5
    for index: int in range(count):
        var x: int = index % columns
        var y: int = index / columns
        slots.append(origin + Vector2(x, y) * spacing)
    return slots
```

Project or reject each slot according to that unit's legal navigation map and
layers. Preserve relative unit-to-slot assignment when possible to reduce path
crossing; for larger groups, use a distance-based assignment rather than array
order.

Spacing should exceed body diameter plus a small margin. Rotate formations to
face travel direction only if the game needs oriented formations.

## Input Tests

- Click each unit and empty terrain at multiple camera zoom levels.
- Click overlapping friendly/enemy units and verify deterministic priority.
- Drag in every direction; normalize the rectangle before testing points.
- Begin or end a drag over UI and confirm the intended owner handles it.
- Delete a selected unit, then issue a command.
- Move a camera while dragging if the game permits it.
- Select a large group and verify unique reachable destinations.
- Right-click unreachable terrain and confirm reject or clamp feedback.
- Pause the game and verify whether selection/commands should still process.

## Common Failures

- Selection misses units when querying areas as bodies or bodies as areas.
- A collision mask of zero or the default all-layers mask produces silent misses
  or irrelevant hits.
- Screen/world coordinate mixing appears only after camera zoom or pan.
- A stale selected reference survives `queue_free()` until a later callback.
- Formation slots are valid on the map but illegal for an agent's navigation
  layers.
- UI consumes nothing because its mouse filter ignores events, or gameplay uses
  `_input()` before UI receives them.
