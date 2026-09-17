# Steering and Interpolation

Use this reference for target-seeking movement, arrival, pursuit approximation, velocity blending, camera smoothing, fixed-duration travel, and rotation smoothing in Godot 4.

## Separate the Layers

Keep these concepts distinct:

- **Intent:** The goal, such as reach or face a target.
- **Desired velocity:** Direction and speed that would satisfy the intent.
- **Steering:** Change from current velocity toward desired velocity.
- **Motion integration:** Applying acceleration or velocity over time.
- **Collision or navigation:** Constraining motion to valid space.

A steering vector does not know about walls. Use navigation for route selection and body motion for collision response.

## Seek and Arrive with CharacterBody2D

This fresh Godot 4 example accelerates toward a target and slows within an arrival radius:

```gdscript
extends CharacterBody2D

@export var target: Node2D
@export_range(1.0, 2000.0, 1.0) var max_speed: float = 260.0
@export_range(1.0, 5000.0, 1.0) var acceleration: float = 900.0
@export_range(1.0, 1000.0, 1.0) var slow_radius: float = 140.0
@export_range(0.0, 100.0, 0.5) var stop_radius: float = 6.0

func _physics_process(delta: float) -> void:
	var desired_velocity: Vector2 = Vector2.ZERO

	if is_instance_valid(target):
		var offset_world: Vector2 = target.global_position - global_position
		var distance: float = offset_world.length()

		if distance > stop_radius:
			var arrival_ratio: float = minf(distance / slow_radius, 1.0)
			var target_speed: float = max_speed * arrival_ratio
			desired_velocity = offset_world / distance * target_speed

	velocity = velocity.move_toward(desired_velocity, acceleration * delta)
	move_and_slide()
```

Important properties:

- Division by `distance` occurs only outside a nonnegative stop radius.
- Speed and acceleration use world units per second and per second squared.
- `move_toward` limits acceleration rather than blending by an arbitrary fraction.
- `move_and_slide` owns collision-aware displacement.

If `slow_radius` is smaller than `stop_radius`, the actor may stop abruptly. Validate exported relationships in `_ready` or a tool script when designers tune them.

## Flee and Blend

Flee reverses the target offset, but should usually be active only inside a threat radius:

```gdscript
static func flee_velocity(
	position_world: Vector2,
	threat_world: Vector2,
	threat_radius: float,
	max_speed: float
) -> Vector2:
	if threat_radius <= 0.0 or max_speed <= 0.0:
		return Vector2.ZERO
	var away_world: Vector2 = position_world - threat_world
	if away_world.is_zero_approx():
		return Vector2.ZERO
	if away_world.length_squared() > threat_radius * threat_radius:
		return Vector2.ZERO
	return away_world.normalized() * max_speed
```

Blend compatible intentions before limiting the result:

```gdscript
static func blend_desired_velocity(
	goal_velocity: Vector2,
	avoidance_velocity: Vector2,
	avoidance_weight: float,
	max_speed: float
) -> Vector2:
	var combined: Vector2 = goal_velocity + avoidance_velocity * avoidance_weight
	return combined.limit_length(maxf(max_speed, 0.0))
```

Weighted blending can cancel into zero or jitter between equally strong demands. Priority steering is often clearer: satisfy immediate collision avoidance first, then use remaining freedom for the goal.

## Bounded Pursuit Approximation

Predicting a moving target's future position can reduce trailing. This is not an exact projectile intercept; it is a bounded steering estimate:

```gdscript
static func predict_target_world_position(
	pursuer_world: Vector2,
	pursuer_speed: float,
	target_world: Vector2,
	target_velocity: Vector2,
	max_prediction_seconds: float
) -> Vector2:
	var distance: float = pursuer_world.distance_to(target_world)
	var safe_speed: float = maxf(absf(pursuer_speed), 0.001)
	var prediction_seconds: float = minf(
		distance / safe_speed,
		maxf(max_prediction_seconds, 0.0)
	)
	return target_world + target_velocity * prediction_seconds
```

Prediction becomes unreliable when the target changes direction frequently, the pursuer cannot follow a direct path, or network velocity is stale. Clamp the horizon and compare against simple seeking in playtests.

## Interpolation Models

### Fixed Progress Between Endpoints

Use fixed endpoints and elapsed time when the move must take a known duration:

```gdscript
static func position_at_time(
	start_world: Vector2,
	end_world: Vector2,
	elapsed_seconds: float,
	duration_seconds: float
) -> Vector2:
	if duration_seconds <= 0.0:
		return end_world
	var progress: float = clampf(elapsed_seconds / duration_seconds, 0.0, 1.0)
	return start_world.lerp(end_world, progress)
```

Do not replace `start_world` with the current position each frame. That creates smoothing, not fixed-duration travel.

### Constant-Speed Approach

Use `move_toward` when distance per second should be constant:

```gdscript
static func approach_at_speed(
	current_world: Vector2,
	target_world: Vector2,
	speed: float,
	delta: float
) -> Vector2:
	return current_world.move_toward(target_world, maxf(speed, 0.0) * delta)
```

This does not overshoot. For a `CharacterBody2D`, prefer velocity plus `move_and_slide` over assigning the returned position.

### Frame-Rate-Independent Smoothing

Use an exponential weight for responsive following in `_process` or variable time steps:

```gdscript
static func smoothing_weight(response_per_second: float, delta: float) -> float:
	var response: float = maxf(response_per_second, 0.0)
	return 1.0 - exp(-response * maxf(delta, 0.0))

func smooth_follow(
	current_world: Vector2,
	target_world: Vector2,
	response_per_second: float,
	delta: float
) -> Vector2:
	var weight: float = smoothing_weight(response_per_second, delta)
	return current_world.lerp(target_world, weight)
```

A larger response converges faster. Because this is asymptotic, snap when an exact terminal state is required.

### Angle Smoothing

Ordinary scalar interpolation can take the long path across the angle wrap. Use `lerp_angle`:

```gdscript
static func smooth_angle(
	current_radians: float,
	target_radians: float,
	response_per_second: float,
	delta: float
) -> float:
	var weight: float = smoothing_weight(response_per_second, delta)
	return lerp_angle(current_radians, target_radians, weight)
```

For 3D orientation, prefer `Quaternion.slerp`, `Basis.slerp`, or `Transform3D.interpolate_with` according to what must be interpolated. Avoid independently interpolating Euler components for compound rotation.

## Steering Edge Cases

- Target and actor occupy the same point.
- Target is freed during a frame.
- Target teleports or crosses a world-wrap boundary.
- Max speed, acceleration, radius, or delta is zero or negative.
- Arrival radius is smaller than collision separation.
- Obstacles make the direct desired velocity impossible.
- Multiple steering terms cancel or rapidly alternate priority.
- CharacterBody floor and slope rules alter the resulting velocity.
- Network snapshots make target velocity noisy.
- Pausing produces a large resumed `delta` in custom time code.

## Unit Tests

- Seek points toward the target and never exceeds max speed.
- Flee is zero outside the radius and points away inside it.
- Arrival target speed decreases monotonically within the slow radius.
- Fixed-duration interpolation returns exact endpoints at progress 0 and 1.
- Constant-speed approach moves the same total distance over equal elapsed time at 30, 60, and 120 steps per second.
- Exponential smoothing reaches nearly the same value for those step rates.
- Angle smoothing takes the short path across `-PI` and `PI`.
- Prediction never exceeds the configured horizon.

Use tolerances for floating-point comparisons.

## Visual Tests

Draw:

- Current velocity in one color and desired velocity in another.
- Slow and stop radii.
- Predicted target point and actual target path.
- Collision normals and post-slide velocity.
- A trail sampled at fixed time intervals.

Run the same fixture at multiple physics tick rates and render frame caps. Test sudden target reversal, teleport, narrow corridors, slopes, moving platforms, and unreachable targets. The path should communicate the chosen motion model rather than accidentally changing with frame rate.
