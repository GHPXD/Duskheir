# Spaces, Vectors, and Trigonometry

Use this reference for coordinate conversions, offsets, distances, directions, alignment, side tests, headings, and periodic motion in Godot 4.

## Coordinate Spaces

Common spaces include:

- **Local:** Relative to a node's own origin and axes.
- **Parent:** The coordinate system in which a node's `position` or `transform` is stored.
- **World or global:** After ancestor transforms are composed.
- **Canvas:** A 2D drawing space that may include canvas transforms.
- **Viewport:** Pixel-like coordinates inside a viewport.
- **Screen:** Operating-system display coordinates; not always the same as viewport coordinates.
- **View or camera:** Coordinates relative to a 3D camera.
- **UV:** Usually normalized texture coordinates, commonly from `(0, 0)` to `(1, 1)`.

Never add values from different spaces. Convert first.

### Typed 2D Conversion

Here `Marker2D` is a direct child, so its `position` belongs to this node's local space:

```gdscript
extends Node2D

@onready var marker: Marker2D = $Marker2D

func place_marker_at_world_point(world_point: Vector2) -> void:
	marker.position = to_local(world_point)

func get_marker_world_point() -> Vector2:
	return to_global(marker.position)
```

Do not pass a node's own `position` to its own `to_global`; that value already belongs to the parent space. Use a point expressed in the receiving node's local space.

### Typed 3D Direction

Subtract world positions to produce a world-space offset. Guard zero before normalization:

```gdscript
static func direction_between_world_points(
	from_world: Vector3,
	to_world: Vector3
) -> Vector3:
	var offset_world: Vector3 = to_world - from_world
	if offset_world.is_zero_approx():
		return Vector3.ZERO
	return offset_world.normalized()
```

A conventional Godot 3D node's forward direction is the negative Z basis axis:

```gdscript
static func world_forward(node: Node3D) -> Vector3:
	return -node.global_transform.basis.z.normalized()
```

Imported models may face a different asset axis. Verify the model and hierarchy instead of applying a compensating sign without evidence.

## Points, Directions, and Velocity

The same vector type can represent different quantities:

- Point: location relative to an origin.
- Offset: displacement from one point to another.
- Direction: orientation, often unit length.
- Velocity: displacement per second.
- Acceleration: velocity change per second.
- Normal: direction perpendicular to a surface.

Useful relationships:

`offset = destination - origin`

`destination = origin + offset`

`velocity = direction * speed`

`new_position = old_position + velocity * delta`

Keep units coherent. If position is pixels and `delta` is seconds, velocity should be pixels per second.

## Length and Distance

Use `length()` when the actual magnitude is needed. Use squared length for comparisons:

```gdscript
static func is_within_radius(
	point_world: Vector2,
	center_world: Vector2,
	radius: float
) -> bool:
	if radius < 0.0:
		return false
	return point_world.distance_squared_to(center_world) <= radius * radius
```

Squared comparisons avoid a square root and are especially useful in frequent proximity checks. Do not compare a squared distance to an unsquared radius.

## Normalize Safely

Normalization preserves direction and changes length to one. A zero vector has no direction.

```gdscript
static func normalized_or_fallback(
	value: Vector2,
	fallback: Vector2 = Vector2.RIGHT
) -> Vector2:
	if value.is_zero_approx():
		return fallback.normalized() if not fallback.is_zero_approx() else Vector2.RIGHT
	return value.normalized()
```

Do not normalize velocity if speed matters. Store or derive direction and speed separately when both are needed.

## Dot Product

For normalized vectors, `a.dot(b)` is:

- Near `1`: same direction.
- Near `0`: perpendicular.
- Near `-1`: opposite direction.

This field-of-view test avoids calculating an angle every frame:

```gdscript
static func is_direction_within_cone(
	forward: Vector2,
	to_target: Vector2,
	half_angle_radians: float
) -> bool:
	if forward.is_zero_approx() or to_target.is_zero_approx():
		return false
	var alignment: float = forward.normalized().dot(to_target.normalized())
	return alignment >= cos(half_angle_radians)
```

Clamp a computed dot value before `acos` when an actual angle is required, because floating-point error can put it slightly outside `[-1, 1]`:

```gdscript
static func angle_between(a: Vector3, b: Vector3) -> float:
	if a.is_zero_approx() or b.is_zero_approx():
		return 0.0
	var cosine: float = clampf(a.normalized().dot(b.normalized()), -1.0, 1.0)
	return acos(cosine)
```

Returning zero for a degenerate input is a policy choice. An assertion, `NAN`, or explicit failure result may be better for critical calculations.

## Cross Product and Side Tests

In 3D, `a.cross(b)` returns a vector perpendicular to both. Operand order determines the sign and direction.

```gdscript
static func triangle_normal(a: Vector3, b: Vector3, c: Vector3) -> Vector3:
	var edge_ab: Vector3 = b - a
	var edge_ac: Vector3 = c - a
	var normal: Vector3 = edge_ab.cross(edge_ac)
	return Vector3.ZERO if normal.is_zero_approx() else normal.normalized()
```

Reversing `b` and `c` reverses the normal. Mesh winding therefore matters.

In 2D, `a.cross(b)` returns a signed scalar. Godot's +Y-down screen convention affects visual intuition, so verify signs with a drawn example:

```gdscript
static func side_of_heading(forward: Vector2, to_target: Vector2) -> float:
	return forward.cross(to_target)
```

Treat values close to zero as collinear rather than relying on exact equality.

## Trigonometry and Angles

Godot script trigonometric functions use radians. Convert only at UI or data boundaries:

```gdscript
var turn_rate_rad_per_second: float = deg_to_rad(120.0)
var display_degrees: float = rad_to_deg(rotation)
```

Use `atan2(y, x)` or `Vector2.angle()` for a heading across all quadrants. Avoid `atan(y / x)`, which loses quadrant information and divides by zero on a vertical direction.

```gdscript
static func heading_to(from_world: Vector2, to_world: Vector2) -> float:
	return (to_world - from_world).angle()
```

In Godot 2D:

- Angle `0` points right.
- `PI / 2` points down.
- `-PI / 2` points up.

Use `lerp_angle` for interpolation across the wrap boundary.

## Circular and Periodic Motion

`Vector2.from_angle` keeps the convention visible:

```gdscript
static func point_on_circle(
	center_world: Vector2,
	radius: float,
	angle_radians: float
) -> Vector2:
	return center_world + Vector2.from_angle(angle_radians) * radius
```

For oscillation, choose amplitude, frequency, phase, and baseline explicitly:

```gdscript
static func oscillate(
	baseline: float,
	amplitude: float,
	frequency_hz: float,
	elapsed_seconds: float,
	phase_radians: float = 0.0
) -> float:
	var angle: float = TAU * frequency_hz * elapsed_seconds + phase_radians
	return baseline + amplitude * sin(angle)
```

Frequency is cycles per second. Angular speed is radians per second. Mixing them causes a factor-of-`TAU` error.

## Reflection and Sliding

A collision normal should be normalized before vector reflection or sliding:

```gdscript
static func reflected_velocity(velocity: Vector2, surface_normal: Vector2) -> Vector2:
	if surface_normal.is_zero_approx():
		return velocity
	return velocity.bounce(surface_normal.normalized())

static func velocity_along_surface(
	velocity: Vector2,
	surface_normal: Vector2
) -> Vector2:
	if surface_normal.is_zero_approx():
		return velocity
	return velocity.slide(surface_normal.normalized())
```

Use the collision response supplied by Godot's body APIs when applicable. These helpers explain or customize behavior; they do not replace continuous collision detection.

## Edge Cases

- Zero vectors and coincident points.
- Negative radii or speeds with unclear semantics.
- Angles outside one turn and wrapping near `PI`.
- Dot values outside the valid inverse-cosine range by tiny error.
- Degenerate triangles with no stable normal.
- Parent scale changing the length of transformed directions.
- Mirrored transforms reversing orientation and winding.
- 2D +Y-down assumptions copied into 3D +Y-up code.
- Very large coordinates losing small movements due to precision.

## Unit and Visual Tests

Unit-test:

- `to_global(to_local(point))` approximately equals `point`.
- Normalized nonzero vectors have length approximately one.
- Squared radius checks agree with ordinary distance checks.
- Cone tests include the exact boundary and reject just outside it.
- Triangle normals reverse when winding reverses.
- Periodic motion returns to its starting phase after one period.

Visualize:

- Local and world axes in different colors.
- Raw offsets and normalized directions at different lengths.
- Field-of-view boundary rays and tested targets.
- Triangle winding and normals.
- Angle zero, positive quarter-turn, and negative quarter-turn in the active coordinate system.
