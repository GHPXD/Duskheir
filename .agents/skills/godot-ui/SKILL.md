---
name: godot-ui
description: "Designs and troubleshoots responsive Godot 4.x Control interfaces using anchors, containers, themes, focus navigation, keyboard and controller input, localization-safe sizing, and accessibility practices. Use when requests mention menus, HUDs, forms, responsive layouts, Control nodes, themes, focus, UI input, or accessible interaction."
license: MIT
metadata:
  author: godot-skills
  version: "1.0.0"
---

# Godot UI

Build interfaces from layout rules, reusable visual resources, and explicit interaction states. Do not solve a responsive layout with a pile of coordinates or make pointer input the only usable path.

## Inspect Before Editing

Complete this inspection before changing scenes, scripts, themes, or project settings.

1. Locate `project.godot` and confirm the project root. Read `config/features`, display base size, stretch mode and aspect, GUI theme settings, localization settings, and Input Map actions.
2. Run the project's established Godot executable with `--version` when available. Use APIs and Inspector property names from that exact Godot 4.x release.
3. Inspect representative UI scenes and scripts. Record root types, `CanvasLayer` use, anchor and container patterns, size flags, unique node names, signal connection style, typed GDScript conventions, scene composition, and naming.
4. Inspect existing `.tres` themes, fonts, font fallbacks, icons, localization files, settings for UI scale, and all interaction states already defined.
5. Run the current UI at its intended design size, smallest supported size, largest practical size, and unusual aspect ratios. Test at least one longer locale if translations exist.
6. Navigate without a mouse. Record initial focus, Tab order, directional neighbors, accept/cancel behavior, focus visibility, and any controls that trap or lose focus.

Do not overwrite project-wide stretch, theme, or input settings to fix one scene without confirming the effect on every existing interface.

## Core Layout Rules

- Use a `Control` root that follows its viewport or UI region.
- Use anchors to place major regions relative to a parent.
- Use containers to arrange dynamic children inside those regions.
- Let a `Container` own the position and size of its direct children. Use size flags, stretch ratios, minimum sizes, theme separation, and wrapper controls instead of fighting it with offsets.
- Use `MarginContainer` for padding, `PanelContainer` for a styled content surface, box containers for rows and columns, and `GridContainer` for a known column count.
- Use a `ScrollContainer` when valid content can exceed available space. Do not shrink text and controls until they technically fit.
- Reserve fixed offsets and minimum sizes for intentional touch targets, readable controls, art boundaries, or layout constraints.
- Keep screen-space game UI under a `CanvasLayer` when it must be independent of a world camera.

## Workflow

1. Define supported window sizes, aspect ratios, UI scale range, input devices, locales, and accessibility options.
2. Sketch semantic regions and reading order before choosing node types.
3. Build the structural tree with anchors and containers using placeholder content. Make resizing work before styling.
4. Add labels, fields, buttons, lists, images, and dynamic content. Use meaningful names and reusable sub-scenes for repeated components.
5. Apply a shared `Theme` at the highest sensible root. Use type variations for roles such as primary, destructive, compact, or heading; use per-node overrides only for true exceptions.
6. Connect semantic signals such as `pressed`, `value_changed`, `item_selected`, and `text_submitted`. Use `_gui_input` only for custom interactions that standard controls cannot express.
7. Define focus entry, order, directional movement, activation, cancel behavior, and focus restoration for every screen and modal.
8. Add accessibility: persistent labels, visible focus, sufficient contrast, non-color cues, scalable text, reduced motion, remappable controls where relevant, and readable status/error feedback.
9. Verify layout, theme states, input routes, localization, and runtime updates together.

## Typed Responsive Example

This example changes a form grid from two columns to one based on the actual available width. It does not guess the device type.

```gdscript
extends Control

@export_range(320.0, 1600.0, 8.0) var compact_width: float = 720.0
@onready var settings_grid: GridContainer = %SettingsGrid

func _ready() -> void:
    resized.connect(_refresh_layout)
    _refresh_layout()

func _refresh_layout() -> void:
    settings_grid.columns = 1 if size.x < compact_width else 2
```

Prefer a layout that naturally reflows through containers. Add a breakpoint script only when the information structure genuinely changes at a constrained width.

## Input and Focus Contract

- Give focus only to interactive controls. Confirm custom controls use an appropriate `focus_mode`.
- Assign initial focus after a screen becomes visible. Restore focus to the opener after closing a dialog or sub-screen.
- Preserve the built-in `ui_*` action semantics unless the project deliberately remaps them. Keep gameplay actions separate from UI navigation.
- Ensure keyboard and controller users can reach every action and can leave every composite control.
- Keep focus indicators visually distinct from hover and pressed states.
- Let standard controls handle text editing, selection, clipboard, IME, and assistive semantics wherever possible.
- Mark decorative overlays with an appropriate `mouse_filter` so they do not block controls beneath them.
- Handle cancel/back at the owning screen or navigation layer, not independently in every child.

## Verification

- Import and parse with the matching editor. When supported, run `<godot> --headless --path <project> --editor --quit` and inspect all output.
- Resize continuously through the supported range. Test landscape, portrait, ultrawide, narrow, and high-DPI scenarios applicable to the project.
- Check for clipping, overlap, zero-sized controls, unwanted scrollbars, excessive empty space, distorted textures, and content that escapes its panel.
- Replace labels with long text, multiline text, empty text, large numbers, right-to-left text where supported, and runtime-generated content.
- Increase UI or content scale and verify that layout reflows rather than crops.
- Test every control state: normal, hover, pressed, focused, disabled, selected, invalid, loading, and empty where applicable.
- Navigate from a fresh screen using only Tab and Shift+Tab, only directional actions, only accept/cancel, and a supported controller.
- Test mouse, touch, and keyboard event propagation where controls overlap or sit over gameplay.
- Verify focus enters modals, stays inside when required, returns to the opener, and remains visible after scene changes.
- Check contrast, focus cues, non-color status indicators, reduced-motion behavior, text alternatives, and error announcements appropriate to the exact engine/platform support.
- Run the integrated scene, not only an isolated component, and report any untested device, locale, or assistive-technology behavior.

## Pitfalls

- Setting offsets on direct children of containers and wondering why the container resets them.
- Using anchors for every item in a list or form instead of letting a container manage dynamic content.
- Using `scale` to make a `Control` fit, which can blur visuals and separate input geometry from layout intent.
- Hard-coding one resolution, one label length, one font metric, or one input device.
- Applying dozens of theme overrides that hide the actual design system.
- Omitting `focus` styles or making focus visually identical to hover.
- Grabbing focus before a hidden screen becomes visible, or never assigning initial focus at all.
- Handling standard button activation in `_input`, causing duplicate activation or bypassing GUI event routing.
- Letting a decorative `Control` with `MOUSE_FILTER_STOP` block the button beneath it.
- Rewriting `LineEdit.text` on every keystroke without preserving caret, selection, paste, and IME behavior.
- Treating tooltips or color alone as the only explanation of a control or error.
- Assuming all Godot 4.x releases expose the same native accessibility API. Inspect the exact version and verify each target platform.

## References

- [Responsive layouts](references/responsive-layouts.md)
- [Themes and visual states](references/themes-and-visual-states.md)
- [Input, focus, and accessibility](references/input-focus-and-accessibility.md)
