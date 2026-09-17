# Transforms, Projection, and Raycasts

Use this reference for local/global conversion, transform composition, pivots, normals, camera projection, screen picking, ray-plane intersections, and Godot 4 physics ray queries.

## Transform Anatomy

`Transform2D` contains two basis axes and an origin. `Transform3D` contains a `Basis` and an origin.

- Origin represents translation.
- Basis represents rotation, scale, and possibly shear.
- A full transform affects points.
- The basis alone affects directions.

### Points and Directions

```gdscript
static func local_point_to_world(node: Node3D, local_point: Vector3) -> Vector3:
	return node.global_transform * local_point

static func local_direction_to_world(
	node: Node3D,
	local_direction: Vector3
) -> Vector3:
	var world_direction: Vector3 = node.global_transform.basis * local_direction
	return Vector3.ZERO if world_direction.is_zero_approx() else world_direction.normalized()
```

Using the full transform on a direction incorrectly adds translation. Using only the basis on a point omits translation.

For ordinary node conversion, prefer the named methods:

```gdscript
static func world_point_in_node_space(node: Node3D, world_point: Vector3) -> Vector3:
	return node.to_local(world_point)

static func node_point_in_world_space(node: Node3D, local_point: Vector3) -> Vector3:
	return node.to_global(local_point)
```

## Composition and Order

For a child transform:

`world_transform = parent_world_transform * child_local_transform`

When applied to a point, the rightmost transform acts first. Do not commute transforms; rotation followed by translation generally differs from translation followed by rotation.

```gdscript
static func composed_world_point(
	parent_world: Transform3D,
	child_local: Transform3D,
	point_child: Vector3
) -> Vector3:
	var child_world: Transform3D = parent_world * child_local
	return child_world * point_child
```

Test order with a point away from the origin. The origin itself often hides rotation mistakes.

### Rotate Around a Pivot

Subtract the pivot, rotate the offset, then add the pivot back:

```gdscript
static func rotate_point_around_y(
	point_world: Vector3,
	pivot_world: Vector3,
	angle_radians: float
) -> Vector3:
	var offset_world: Vector3 = point_world - pivot_world
	var rotation_basis: Basis = Basis(Vector3.UP, angle_radians)
	return pivot_world + rotation_basis * offset_world
```

For an actual hierarchy, a pivot `Node3D` parent is often clearer and easier to inspect than repeated manual math.

## Inverses and Scale

Use `affine_inverse()` when a transform may contain scale or shear:

```gdscript
static func world_to_local_affine(
	transform_world: Transform3D,
	point_world: Vector3
) -> Vector3:
	return transform_world.affine_inverse() * point_world
```

`inverse()` assumes an orthonormal basis. A basis with a zero scale component cannot be inverted. Guard authoring data and runtime animation from singular scales.

## Transforming Normals

For rotation-only or uniform-scale transforms, transforming and normalizing with the basis is sufficient. Non-uniform scale requires the inverse transpose so the result stays perpendicular to the surface:

```gdscript
static func local_normal_to_world(
	transform_world: Transform3D,
	local_normal: Vector3
) -> Vector3:
	if local_normal.is_zero_approx():
		return Vector3.ZERO
	var normal_basis: Basis = transform_world.basis.inverse().transposed()
	return (normal_basis * local_normal).normalized()
```

This requires an invertible basis. Mirroring can reverse orientation; verify winding, culling, and expected normal direction visually.

## Projection Concepts

A typical 3D rendering chain is:

`local -> world -> camera/view -> clip -> normalized device -> viewport`

Godot's camera handles this chain. Use its methods rather than rebuilding the projection matrix for ordinary gameplay:

- `unproject_position(world_point)` maps a world point to viewport coordinates.
- `is_position_behind(world_point)` checks whether UI should be hidden for a point behind the camera.
- `project_position(screen_point, depth)` maps a viewport point to a world point at a camera depth.
- `project_ray_origin(screen_point)` and `project_ray_normal(screen_point)` construct a world ray for picking.

The naming follows Godot's API and may differ from terminology in other engines.

### World Label Position

```gdscript
static func world_to_viewport_if_visible(
	camera: Camera3D,
	world_point: Vector3
) -> Variant:
	if camera.is_position_behind(world_point):
		return null
	return camera.unproject_position(world_point)
```

Returning `Variant` allows either a `Vector2` or `null`. A point in front can still be outside the viewport, so check the viewport rectangle when that matters.

## Ray Geometry

A ray can be written as:

`point(t) = origin + direction * t`, where `t >= 0`

A physics query uses a finite segment from `origin` to `origin + direction * max_distance`.

### Screen Ray to a Horizontal Plane

This calculation finds a world point on the plane `y = plane_y` without querying physics:

```gdscript
static func screen_ray_to_y_plane(
	camera: Camera3D,
	screen_point: Vector2,
	plane_y: float
) -> Variant:
	var ray_origin: Vector3 = camera.project_ray_origin(screen_point)
	var ray_direction: Vector3 = camera.project_ray_normal(screen_point)

	if absf(ray_direction.y) <= 0.000001:
		return null

	var distance_along_ray: float = (plane_y - ray_origin.y) / ray_direction.y
	if distance_along_ray < 0.0:
		return null

	return ray_origin + ray_direction * distance_along_ray
```

Near-parallel rays magnify error. If a maximum interaction distance exists, reject larger values of `distance_along_ray`.

## Physics Raycast from a Camera

Run direct-space queries from a physics-safe point, typically `_physics_process`:

```gdscript
static func raycast_from_camera(
	camera: Camera3D,
	screen_point: Vector2,
	max_distance: float,
	collision_mask: int = 0xFFFFFFFF,
	include_areas: bool = false
) -> Dictionary:
	if max_distance <= 0.0:
		return {}

	var ray_origin: Vector3 = camera.project_ray_origin(screen_point)
	var ray_direction: Vector3 = camera.project_ray_normal(screen_point)
	var ray_end: Vector3 = ray_origin + ray_direction * max_distance

	var query: PhysicsRayQueryParameters3D = PhysicsRayQueryParameters3D.create(
		ray_origin,
		ray_end,
		collision_mask
	)
	query.collide_with_areas = include_areas

	var space_state: PhysicsDirectSpaceState3D = camera.get_world_3d().direct_space_state
	return space_state.intersect_ray(query)
```

An empty dictionary means no hit. A hit dictionary provides data such as collider, position, normal, face index, RID, and shape index according to the collider and backend.

To exclude the querying body, assign its RID:

```gdscript
static func exclude_body_from_ray(
	query: PhysicsRayQueryParameters3D,
	body: CollisionObject3D
) -> void:
	var excluded: Array[RID] = [body.get_rid()]
	query.exclude = excluded
```

Collision masks are physics layers, not the camera's visual cull mask.

## Perspective and Orthographic Cameras

Always call both ray helper methods. With perspective projection, rays spread from the camera. With orthographic projection, ray directions are parallel while origins vary across the viewport. Assuming every ray begins at `camera.global_position` breaks orthographic picking.

Camera near and far planes affect rendering precision and visibility, not the explicit `max_distance` of a physics query.

## 2D Equivalents

The same concepts apply with:

- `Transform2D`, `Node2D.to_local`, and `Node2D.to_global`.
- `PhysicsRayQueryParameters2D`.
- `World2D.direct_space_state.intersect_ray`.
- Global canvas points rather than 3D camera rays for ordinary 2D picking.

Account for CanvasLayer and viewport transforms when UI and world canvases differ.

## Edge Cases

- Point exactly at the camera or behind it.
- Viewport resized after caching projection-dependent values.
- SubViewport coordinates passed as root viewport coordinates.
- Orthographic versus perspective assumptions.
- Ray parallel or nearly parallel to a plane.
- Ray starts inside a collider; `hit_from_inside` policy matters.
- Back-face behavior on concave shapes.
- Areas excluded by default.
- Self-collision not excluded.
- Visual and physics geometry disagree.
- Non-uniform, negative, or zero ancestor scale.
- Reparenting changes local transform interpretation.
- Large ray distances and large worlds reduce precision.

## Unit Tests

- Local/world round trips under translation, rotation, uniform scale, and non-uniform scale.
- Composition against hand-selected points where order produces clearly different results.
- Direction transforms remain translation-independent.
- Transformed normals remain perpendicular to transformed tangents.
- Screen-to-plane returns null for parallel and behind-origin intersections.
- Camera ray center aligns with expected forward direction.
- Collision masks include and exclude fixtures correctly.
- No-hit, exact-boundary hit, inside-start, area, body, and excluded-body queries.

## Visual Tests

- Draw each node's basis axes and origin.
- Draw a point before and after every transform stage.
- Draw surface tangents and normals under non-uniform scale.
- Draw rays from viewport center and corners for both camera projections.
- Mark hit position, normal, collider name, layer, and distance.
- Resize the viewport and test SubViewports, stretch modes, and UI overlays.

Keep the visualization in the same space as the values it represents; a correct vector drawn through the wrong transform looks like incorrect math.
