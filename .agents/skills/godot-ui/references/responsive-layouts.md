# Responsive Layouts

Responsive Godot UI comes from clear ownership of geometry. Anchors place a region; containers arrange the region's content; theme constants provide reusable spacing.

## Geometry Ownership

For each `Control`, identify exactly one owner of its rectangle:

- A plain parent plus anchors and offsets.
- A direct parent `Container` plus size flags, stretch ratio, and minimum size.
- Custom layout code, only when the built-in system cannot express the behavior.

Do not mix all three casually. A container recalculates direct-child geometry and will overwrite manual positions.

## Stable Page Skeleton

```text
Screen (Control, Full Rect)
`-- PageMargin (MarginContainer, Full Rect)
    `-- PageColumn (VBoxContainer)
        |-- Header
        |-- Body (expands)
        `-- Footer
```

The root follows the viewport. The margin owns the page gutter. The column owns vertical distribution. `Body` receives the expand flag so the header and footer keep their minimum sizes.

For a styled card, use:

```text
PanelContainer
`-- MarginContainer
    `-- VBoxContainer
        |-- Heading
        |-- Content
        `-- Actions
```

Each node has one role: surface, padding, then flow. This makes theme changes and content growth predictable.

## Container Selection

- `HBoxContainer` and `VBoxContainer`: ordered rows and columns.
- `GridContainer`: repeated cells with a known column count. It does not provide arbitrary cell spanning.
- `CenterContainer`: center one visual or content branch without manual offsets.
- `MarginContainer`: theme-driven outer padding.
- `PanelContainer`: a themed background that participates in content sizing.
- `ScrollContainer`: bounded viewport for content whose valid size can exceed the screen.
- `HFlowContainer` or `VFlowContainer`: wrapping sets of controls when the inspected Godot version supports the required behavior.
- `AspectRatioContainer`: preserve media or preview proportions.

Use wrapper `Control` nodes sparingly when one element must escape a container's immediate layout. Document why the wrapper exists.

## Size Flags and Minimum Size

Use size flags to answer two questions:

- May this child receive extra space on an axis?
- Where should it sit if it does not fill that space?

Use stretch ratios only among siblings that all expand on the same axis. Ratios divide flexible space; they do not override a child's minimum size.

`custom_minimum_size` is a lower bound, not a final rectangle. Use it for legitimate constraints such as a readable field height, a touch target, or a preview that must not collapse. Excessive minimum sizes force overflow and defeat responsiveness.

## Responsive Gutters

This script keeps page padding proportional within sensible limits. Attach it to a full-rect `MarginContainer` and preserve the project's spacing values.

```gdscript
extends MarginContainer

@export_range(0.0, 0.2, 0.005) var gutter_ratio: float = 0.04
@export_range(0, 128, 1) var min_gutter: int = 16
@export_range(0, 256, 1) var max_gutter: int = 64

func _ready() -> void:
    resized.connect(_update_gutter)
    _update_gutter()

func _update_gutter() -> void:
    var gutter: int = clampi(roundi(size.x * gutter_ratio), min_gutter, max_gutter)
    add_theme_constant_override(&"margin_left", gutter)
    add_theme_constant_override(&"margin_top", gutter)
    add_theme_constant_override(&"margin_right", gutter)
    add_theme_constant_override(&"margin_bottom", gutter)
```

If many screens use this behavior, move the spacing into theme constants or a reusable component instead of repeating scripts.

## Images and Art

- Select `TextureRect` expand and stretch modes intentionally.
- Preserve aspect ratio for logos, portraits, previews, and screenshots unless cropping is part of the design.
- Use nine-patch assets or `StyleBoxTexture` for scalable framed artwork.
- Set texture filtering according to the art style and actual scale behavior.
- Avoid using a high-resolution bitmap as a full UI layout.
- Confirm icons remain recognizable in disabled, focused, selected, and high-contrast states.

## Text and Localization

- Let labels calculate their natural minimum size.
- Enable wrapping for prose, errors, descriptions, and dynamic messages.
- Define what happens to single-line titles: grow, clip, ellipsize, or scroll.
- Include font fallbacks for every supported script and symbol set.
- Test translated strings that are substantially longer than the source language.
- Test bidirectional and right-to-left layout if those locales are supported.
- Avoid concatenating translated fragments whose word order may change.

## Stretch and Content Scale

Read the current project settings before recommending changes.

- `canvas_items` generally renders 2D and UI at target resolution while scaling layout from a base size.
- `viewport` renders into the base viewport first, then scales the result; it can suit low-resolution or pixel-art presentation.
- A disabled stretch mode can be appropriate for resizable non-game desktop applications that manage minimum window size and UI scale directly.
- Stretch aspect determines whether the project keeps, expands, or fixes one dimension. Choose it from content behavior, not habit.

Do not change stretch settings to hide a broken container tree. First prove which problem belongs to viewport scaling and which belongs to layout.

## Responsive Test Matrix

At minimum, test:

- Smallest supported width and height.
- Design size.
- Largest common window.
- Narrow portrait and wide landscape.
- Long labels and multiline errors.
- Empty lists and maximum expected list contents.
- Minimum and maximum UI scale.
- Runtime show/hide of optional sections.
- Font fallback and translated text.

Watch the remote scene tree and each control's minimum size while debugging. A surprising ancestor minimum often explains why a whole screen refuses to shrink.
