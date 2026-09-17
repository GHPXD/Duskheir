# Themes and Visual States

Use themes as the interface's visual contract. Layout scenes should describe structure and meaning; theme resources should describe repeated typography, spacing, colors, icons, and state surfaces.

## Theme Scope

Apply a shared `Theme` at the highest root that should share a visual language. Descendant controls inherit it unless a nearer theme or local override takes priority.

Choose deliberately among:

- A project-wide default for a single product-wide system.
- A root theme per application shell or game UI layer.
- A subtree theme for a deliberately distinct embedded surface.
- A theme type variation for semantic roles within one control class.
- A local override for a one-off exception.

Repeated local overrides are a signal that the style belongs in the theme.

## Define Every Meaningful State

Interactive controls may need:

- Normal.
- Hover.
- Pressed.
- Focused.
- Disabled.
- Checked or selected.
- Read-only.
- Invalid or warning, usually through a type variation or companion message.

Do not rely only on background color. Combine color with border, icon, label, weight, underline, or another visible cue. Focus must remain clear when hover is absent.

## StyleBox Strategy

- `StyleBoxFlat` is suitable for procedural fills, borders, corner radii, content margins, and shadows.
- `StyleBoxTexture` supports art-driven surfaces and nine-slice scaling.
- `StyleBoxEmpty` deliberately removes a surface while preserving the state slot.
- Shared style resources update every user. Duplicate before changing a single semantic variant.

Use content margins in the style or a `MarginContainer` consistently. Avoid hidden padding split unpredictably between both.

## Type Variations

Create named roles such as `PrimaryButton`, `DangerButton`, `ToolbarButton`, `SectionHeading`, or `ValidationMessage`. Set the variation's base type and override only what differs.

Semantic variation names survive palette changes better than names such as `BlueButton` or `LargeRedText`.

## Runtime Theme Switching

Assign authored theme resources in the Inspector. Keep preference storage outside this view component.

```gdscript
extends Control

signal theme_mode_applied(mode: StringName)

@export var light_theme: Theme
@export var dark_theme: Theme

func apply_theme_mode(mode: StringName) -> void:
    assert(is_instance_valid(light_theme))
    assert(is_instance_valid(dark_theme))

    match mode:
        &"dark":
            theme = dark_theme
        _:
            theme = light_theme

    theme_mode_applied.emit(mode)
```

If the application supports `system`, query it through APIs available in the exact Godot release and platform, then resolve to an authored light or dark resource. Always provide a user override because system detection is not uniform across every Godot 4.x target.

## Typography

- Define base font, fallbacks, sizes, outline, and spacing in the theme where possible.
- Keep a small role scale: body, caption, heading, display, and control text are often enough.
- Verify glyph coverage before shipping a custom font.
- Preserve readable line height and avoid all-caps text for long content.
- Test font oversampling and UI scale at the project's actual stretch mode.
- Do not rasterize important labels into textures.

## Color and Contrast

- Test text and icons against the actual state background, not an isolated swatch.
- Keep disabled content distinguishable without making required information unreadable.
- Verify focus and selection under both light and dark modes.
- Provide a high-contrast option when the product requires it.
- Check color-vision-independent cues for success, warning, error, teams, and item rarity.
- Evaluate translucent surfaces over every background they can cover.

## Motion

Theme changes and state transitions can use animation, but motion is feedback rather than layout logic.

- Keep durations consistent by role.
- Do not delay activation until a decorative animation ends.
- Avoid large spatial movement for routine focus or hover.
- Provide a reduced-motion path that removes or shortens nonessential movement.
- Stop or reconcile tweens when controls leave the tree or state changes rapidly.

## Theme Verification

- Create a component gallery containing every themed control and state.
- Compare normal, hover, pressed, focus, disabled, selected, invalid, and read-only states side by side.
- Test each theme mode at minimum and maximum UI scale.
- Check long text, fallback glyphs, icons, and right-to-left presentation.
- Inspect reused `StyleBox`, font, and theme resources before editing to avoid unintended global changes.
- Run real screens as well as the gallery; background context can change perceived contrast and spacing.

## Pitfalls

- Styling each scene independently until similar controls no longer match.
- Using one mutable `StyleBoxFlat` for states that must diverge.
- Removing focus visuals to achieve a cleaner screenshot.
- Making disabled controls differ only by a tiny opacity change.
- Encoding semantic meaning only in hue.
- Switching theme by rebuilding the scene and losing focus or form state.
- Assuming an editor theme or operating-system palette automatically applies to the game UI.
