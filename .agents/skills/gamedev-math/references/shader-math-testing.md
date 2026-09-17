# Shader Math and Testing

Use this reference for normalized ranges, remapping, masks, UV effects, normals, CPU-to-GPU parameters, floating-point safety, and visual verification in Godot 4.

## Think in Spaces, Ranges, and Stages

Before writing shader math, identify:

- Shader type: `canvas_item`, `spatial`, particles, sky, or fog.
- Stage: vertex, fragment, or light.
- Coordinate space of every vector.
- Expected numeric range of every input and output.
- Whether data is linear color, source color, a normal, a position, or a direction.
- Renderer and target hardware.

UV coordinates are not world positions. A normal is not a color even if both use three components. Convert intentionally.

## Core Range Operations

Useful operations include:

- `clamp(x, low, high)`: bound a value.
- `mix(a, b, t)`: interpolate or extrapolate by `t`.
- `step(edge, x)`: hard threshold.
- `smoothstep(low, high, x)`: smooth threshold over an ordered interval.
- `fract(x)`: repeating fractional phase.
- `dot(a, b)`: alignment, projection, and simple lighting.
- `length(v)` and `distance(a, b)`: radial masks and falloff.
- `normalize(v)`: remove magnitude, with a zero-length safety policy.

Use an explicit remap in typed GDScript when preparing or testing values on the CPU:

```gdscript
static func remap_clamped(
	value: float,
	input_min: float,
	input_max: float,
	output_min: float,
	output_max: float
) -> float:
	if is_equal_approx(input_min, input_max):
		return output_min
	var ratio: float = clampf(
		(value - input_min) / (input_max - input_min),
		0.0,
		1.0
	)
	return lerpf(output_min, output_max, ratio)
```

Document the degenerate-range policy. Returning `output_min` is not mathematically unique, but it is deterministic and testable.

## Fresh Canvas Shader Example

This Godot 4 shader blends a radial energy tint while preserving texture alpha:

```glsl
shader_type canvas_item;

uniform float energy : hint_range(0.0, 1.0) = 0.0;
uniform vec4 glow_color : source_color = vec4(0.15, 0.85, 1.0, 1.0);

void fragment() {
	vec4 texel = texture(TEXTURE, UV);
	float radius = distance(UV, vec2(0.5));
	float center_mask = 1.0 - smoothstep(0.1, 0.65, radius);
	float blend_weight = clamp(center_mask * energy * glow_color.a, 0.0, 1.0);
	vec3 tinted_rgb = mix(texel.rgb, glow_color.rgb, blend_weight);
	COLOR = vec4(tinted_rgb, texel.a);
}
```

The effect operates in UV space, so a non-square texture produces an ellipse in pixel space. If a circular pixel-space effect is required, correct for aspect ratio with a uniform or use a world/view-space formulation.

### Set the Uniform from Typed GDScript

Duplicate a shared material when each instance needs independent state:

```gdscript
extends Sprite2D

var runtime_material: ShaderMaterial

func _ready() -> void:
	var source_material: ShaderMaterial = material as ShaderMaterial
	assert(source_material != null, "Sprite requires a ShaderMaterial")
	runtime_material = source_material.duplicate() as ShaderMaterial
	material = runtime_material

func set_energy(value: float) -> void:
	runtime_material.set_shader_parameter("energy", clampf(value, 0.0, 1.0))
```

Uniform names are case-sensitive. Cache gameplay state on the CPU rather than reading shader parameters back every frame.

## Vertex Waves and Units

Periodic shader motion follows the same amplitude, frequency, and phase model as script. This example moves vertices horizontally based on UV height:

```glsl
shader_type canvas_item;

uniform float amplitude_pixels : hint_range(0.0, 24.0) = 4.0;
uniform float cycles_per_texture : hint_range(0.0, 8.0) = 2.0;
uniform float cycles_per_second : hint_range(-4.0, 4.0) = 0.5;

void vertex() {
	float phase = UV.y * cycles_per_texture * 6.28318530718;
	phase += TIME * cycles_per_second * 6.28318530718;
	VERTEX.x += sin(phase) * amplitude_pixels;
}
```

The amplitude is in canvas vertex units, usually pixels for an unscaled 2D item. Parent scale changes the visible displacement. `TIME` is presentation time, not deterministic gameplay state.

## Normals and Lighting Math

The dot product between normalized surface and light directions gives cosine-weighted facing:

`diffuse = max(dot(normal, light_direction), 0)`

Both vectors must be in the same space. A normal transformed by non-uniform scale needs inverse-transpose treatment before normalization. In a Godot spatial shader, inspect the exact built-in space for the stage and render mode rather than assuming world space.

Normals encoded in textures usually require unpacking from `[0, 1]` into `[-1, 1]` before normalization:

`normal = normalize(encoded * 2 - 1)`

Do not apply source-color conversion hints to data textures such as normals, roughness, or masks. Use source-color hints for albedo and other actual color textures.

## CPU and GPU Differences

- GDScript `float` uses higher precision than the common 32-bit vector and shader float path.
- Shader local variables are not guaranteed to initialize to zero; assign them before use.
- Exact floating-point equality is fragile on both CPU and GPU.
- GPU derivatives, filtering, precision, and branch execution can vary by hardware and renderer.
- Matrix storage and indexing are column-oriented in Godot shaders.
- Uniform updates cross a CPU/GPU boundary; avoid unnecessary per-object churn.
- Reading global shader parameter values can force synchronization; retain a CPU copy instead.

## Shader Edge Cases

- `smoothstep` edges equal or reversed.
- Division by zero or near-zero uniform.
- Normalizing a zero vector.
- `pow` with a negative base and non-integer exponent.
- Square-root or inverse-trig input outside its valid domain.
- UVs outside `[0, 1]` under repeat or clamp sampling.
- Transparent texels retaining bright RGB and causing fringes.
- Non-square textures and resized viewports.
- Shared materials receiving per-instance state.
- `TIME` rollover, pause behavior, or nondeterminism.
- HDR values intentionally outside `[0, 1]` being clamped accidentally.
- Compatibility renderer lacking a feature used by Forward+ or Mobile.
- Dynamic loops, texture array indexing, or precision behaving differently on target GPUs.

## Unit-Test CPU Math

Keep reusable range and geometry math in typed GDScript when the CPU also needs it. Test:

```gdscript
func test_remap_clamped() -> void:
	assert(is_equal_approx(remap_clamped(5.0, 0.0, 10.0, -1.0, 1.0), 0.0))
	assert(is_equal_approx(remap_clamped(-5.0, 0.0, 10.0, 0.0, 1.0), 0.0))
	assert(is_equal_approx(remap_clamped(15.0, 0.0, 10.0, 0.0, 1.0), 1.0))
	assert(is_equal_approx(remap_clamped(2.0, 3.0, 3.0, 7.0, 9.0), 7.0))
```

Also test finite outputs for extreme, negative, and degenerate inputs. A unit test cannot prove a GPU image is correct, but it can verify shared equations and uniform preparation.

## Visual Test Matrix

Create a minimal fixture with deterministic textures, geometry, camera, environment, and uniforms. Capture baselines for:

- Uniform minimum, midpoint, and maximum.
- UV center, edges, and corners.
- Square, wide, and tall textures or viewports.
- Opaque, partially transparent, and fully transparent texels.
- Scaled, rotated, and mirrored nodes.
- Forward+, Mobile, and Compatibility when supported.
- Desktop and target mobile GPU where relevant.
- HDR enabled and disabled when relevant.
- Pause, low frame rate, and long-running time-dependent effects.

Use tolerances for image comparison because antialiasing and GPU precision can vary. Pair snapshots with numeric debug views that render masks, normals, UVs, or intermediate values directly as colors.

## Debug Views

Temporarily output one quantity at a time:

- `COLOR = vec4(vec3(mask), 1.0)` for a scalar mask.
- Encode a signed vector as `vector * 0.5 + 0.5` for display.
- Render UV as red and green channels.
- Render non-finite or out-of-range conditions in a conspicuous fallback color.

Remove debug branches from production shaders when they add measurable cost, or guard them with a compile-time/render-mode strategy used by the project.

## Verification Checklist

- Every value has a named space and expected range.
- Degenerate divisions, normalization, and domains have explicit policies.
- CPU uniforms are typed, bounded, and use exact parameter names.
- Per-instance state does not accidentally mutate a shared material.
- Color and data textures use appropriate import and uniform hints.
- The shader compiles and renders on every supported renderer.
- Numeric helper tests pass.
- Visual baselines cover geometry, alpha, aspect, precision, and time edge cases.
- Performance is measured on target hardware rather than inferred from shader length.
