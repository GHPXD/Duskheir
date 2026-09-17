---
name: gamedev-math
description: "Applies game-development math in Godot 4 with typed GDScript examples for coordinate spaces, vectors, trigonometry, steering, interpolation, transforms, projection, raycasts, and shader math. Use when a request mentions direction, distance, angles, aiming, pursuit, smooth movement, transform order, local versus global coordinates, screen-to-world picking, ray queries, normals, UV math, or debugging spatial and visual calculations."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Gamedev Math

Use this skill to solve spatial and visual problems by naming coordinate spaces, units, and invariants before writing equations. Prefer Godot's built-in vector, transform, camera, and physics APIs over hand-written component math unless the task is specifically educational or requires a custom derivation.

## Required First Pass

Inspect the project before changing code:

- Read `project.godot` for the exact Godot version, renderer, physics tick rate, stretch settings, units, and input configuration.
- Identify whether the problem is 2D, 3D, UI, shader, physics, navigation, or a conversion between them.
- Read the relevant node hierarchy and transforms, including scaled or mirrored ancestors.
- Locate whether state updates in `_process`, `_physics_process`, animation, a tween, a shader, or a server callback.
- Confirm whether values are positions, directions, velocities, accelerations, angles, normalized ratios, pixels, world units, seconds, or units per second.
- Reproduce the problem with debug drawing or logged values before replacing the math.

Do not assume a Vector2 is in world space or that a Vector3 represents a point. The type stores components; surrounding code gives those components meaning.

## Problem-Solving Workflow

### 1. State the Invariant

Write the desired behavior in measurable terms, for example:

- The actor reaches the target at no more than 180 pixels per second.
- A hit is valid when the target direction lies within 25 degrees of forward.
- Converting a point from local to world and back returns the original point within tolerance.
- A screen pick tests physics bodies on layers 1 and 4 up to 500 world units away.
- Camera smoothing has nearly the same response at 30, 60, and 120 frames per second.

### 2. Label Spaces and Units

Annotate variables in names or nearby comments when ambiguity is likely:

- `target_world`, `offset_local`, `cursor_viewport`, `normal_view`.
- `speed_px_per_second`, `turn_rate_rad_per_second`, `duration_seconds`.

Draw the conversion chain. Convert once at a clear boundary, perform related math in one space, then convert the result if needed.

Use [Spaces, Vectors, and Trigonometry](references/spaces-vectors-trig.md) for positions, directions, distances, dot/cross products, headings, and angle conventions.

### 3. Choose the Right Primitive

- Use `length_squared()` for threshold comparisons that do not need the distance.
- Normalize only when magnitude must be removed, and guard the zero vector.
- Use a dot product for alignment or field-of-view tests.
- Use a cross product for perpendicular directions, surface normals, or side tests.
- Use `atan2` or vector angle helpers for a full signed heading.
- Use transforms for parent/local conversions and composed rotation, scale, and translation.
- Use camera projection helpers for viewport/world conversion.
- Use physics queries for collisions; a mathematical ray alone does not inspect the physics world.

### 4. Choose Time Behavior

Distinguish:

- Constant speed: use `move_toward` with a rate times `delta`.
- Fixed-duration travel: derive a normalized progress value and interpolate from fixed endpoints.
- Responsive smoothing: use a frame-rate-independent exponential weight.
- Physics movement: set velocity or forces in `_physics_process` and let the body API resolve collisions.
- Authored easing: use a Tween or curve when timing, not physical response, is the requirement.

Use [Steering and Interpolation](references/steering-interpolation.md).

### 5. Use Engine Space Conversions

Prefer `Node2D.to_local`, `Node2D.to_global`, `Node3D.to_local`, `Node3D.to_global`, `Transform2D`, `Transform3D`, and `Basis`. Keep points and directions distinct: translation affects a point but not a direction.

For screen projection, picking, transform order, normals, and ray queries, use [Transforms, Projection, and Raycasts](references/transforms-projection-raycasts.md).

### 6. Make the Math Observable

Add temporary instrumentation:

- Draw origins, axes, vectors, target radii, rays, normals, and collision points.
- Display units and spaces in debug labels.
- Log precondition failures and non-finite values once, not every frame.
- Freeze or single-step time around boundary cases.
- Use fixed random seeds and fixed inputs for reproducibility.

Remove or gate debug output after verification.

### 7. Verify at Boundaries

Test zero, near-zero, negative, exact-boundary, very large, and mirrored values. Compare approximate floating-point values with `is_equal_approx`, vector `is_equal_approx`, or an explicit tolerance rather than exact equality.

Use [Shader Math and Testing](references/shader-math-testing.md) for remapping, masks, normals, CPU-to-GPU parameters, renderer checks, and visual baselines.

## Correctness Rules

- In Godot 2D, +X points right and +Y points down. Positive vector angles proceed from right toward down.
- In conventional Godot 3D nodes, +X is right, +Y is up, and forward is -Z.
- Most script trigonometric functions use radians. Inspector angle fields may display degrees.
- Multiplication order matters. In `parent_transform * local_transform * point`, the rightmost operation affects the point first.
- A transform with non-uniform scale needs an affine inverse for general point conversion.
- Surface normals under non-uniform scale require inverse-transpose treatment.
- Interpolation weight is a ratio, not a speed. Clamp it only when extrapolation is not intended.
- Physics queries use global coordinates and collision masks, and should run at a safe point in the physics update.
- Shaders commonly use 32-bit floats and different coordinate spaces from script. Initialize local shader variables explicitly.

## Verification Checklist

### Unit Tests

- Round-trip every coordinate conversion.
- Test vector identities and zero-vector guards.
- Test angle wrap near `-PI` and `PI`.
- Simulate movement for equal elapsed time at several step sizes.
- Test transform composition against a known point and direction.
- Test ray-plane parallel, behind-origin, exact-hit, and no-hit cases.
- Test shader helper equivalents at minimum, midpoint, maximum, and degenerate ranges.

### Visual Tests

- Draw local axes after parent rotation, scale, and mirroring.
- Draw steering velocity, desired velocity, arrival radius, and collision response.
- Draw camera rays and hit normals from center, corners, and resized viewports.
- Compare perspective and orthographic cameras.
- Capture shader baselines at representative uniforms, resolutions, renderers, and color modes.
- Run at low and high frame rates and with slow motion or pause where supported.

## Common Edge Cases

- Coincident points produce an undefined direction.
- Nearly parallel vectors make angle and intersection calculations unstable.
- A zero or near-zero scale makes a transform non-invertible.
- Reparenting changes local transforms while preserving or changing world transforms depending on the operation.
- Interpolating from the current value each frame is not fixed-duration interpolation.
- Directly setting a physics body's position can tunnel or bypass collision response.
- Orthographic camera rays have different origins across the viewport.
- A ray starts inside a collider or strikes a back face.
- Collision layers differ from visual render layers.
- Large world coordinates lose floating-point precision.
- Shared ShaderMaterials make one instance's uniform change affect others.
- Transparent, HDR, mobile, and Compatibility rendering expose different shader assumptions.

## Expected Output

When solving a math task, provide:

1. The spaces, units, update loop, and invariant.
2. The smallest correct equation or Godot API operation.
3. Typed Godot 4 GDScript integrated with the project's node types.
4. Guards for degenerate inputs and transform assumptions.
5. Unit and visual verification steps.
6. Any remaining version, renderer, physics, or precision assumptions.
