# Input, Focus, and Accessibility

An interface is complete only when users can discover, reach, understand, activate, and recover from every action with the supported input methods.

## Prefer Semantic Controls and Signals

Use standard controls whenever they match the interaction:

- `Button` or `BaseButton` signals for activation and toggles.
- `LineEdit.text_submitted` for single-line submission.
- `Range.value_changed` for sliders and spin boxes.
- `OptionButton.item_selected` for a compact selection.
- `ItemList.item_selected` or `Tree` signals for collections.
- `TabContainer` for tab semantics when its behavior matches the design.

Standard controls already implement substantial pointer, keyboard, text editing, focus, and theme behavior. A custom-drawn control must recreate and verify those contracts.

## Input Routing

Use the narrowest callback that owns the behavior:

- A control's semantic signal for normal activation.
- `_gui_input(event)` for a custom interaction inside one control.
- `_unhandled_input(event)` for screen-level shortcuts or cancel behavior after GUI controls had a chance to consume the event.
- `_input(event)` only for truly global capture that must run before normal routing.

Call `accept_event()` in `_gui_input` or mark the viewport input handled only when that layer owns the event. Otherwise, allow propagation.

Check `mouse_filter` on overlays:

- `STOP` receives and blocks pointer events.
- `PASS` receives and permits bubbling through the control ancestry.
- `IGNORE` does not receive pointer events and allows controls behind it to be targeted.

## Focus Lifecycle

Every screen should define:

- Entry focus when opened by keyboard or controller.
- A predictable Tab order.
- Directional neighbors for spatial layouts where automatic selection is ambiguous.
- Focus containment for modal dialogs when required.
- Focus restoration to the invoking control when a child screen closes.
- A fallback if the focused control is hidden, disabled, or freed.

```gdscript
extends Control

signal close_requested

@export var initial_focus: Control

func _ready() -> void:
    assert(is_instance_valid(initial_focus))

func focus_screen() -> void:
    initial_focus.grab_focus()

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed(&"ui_cancel"):
        close_requested.emit()
        get_viewport().set_input_as_handled()
```

Call `focus_screen()` after the screen becomes visible. A navigation owner should remember the opener and restore its focus after `close_requested` is handled.

## Focus Graph Review

Run this review without touching the mouse:

1. Open the screen and confirm focus is visible immediately.
2. Press Tab through every control, then Shift+Tab back to the start.
3. Use directional actions through rows, columns, grids, sliders, and tabs.
4. Activate controls with `ui_accept`.
5. Leave text editing and nested controls without losing the overall route.
6. Open and close every modal, popup, and sub-screen.
7. Disable or hide the currently focused control and confirm a valid fallback.
8. Repeat with each supported controller.

Do not repurpose built-in `ui_*` actions for gameplay while a menu also depends on them.

## Keyboard Shortcuts

- Use `Shortcut` resources or the project's existing command system for visible actions.
- Display platform-appropriate shortcut labels where discoverability matters.
- Avoid intercepting common text-editing shortcuts while a text control has focus.
- Provide menu or button access to important commands; a shortcut must not be the only route.
- Confirm repeat behavior for held keys is intentional.
- Test keyboard layouts and platform command modifiers supported by the product.

## Form Feedback

- Pair each field with a persistent visible label; placeholder text is not a label.
- Validate without destroying valid intermediate typing states.
- Preserve caret, selection, clipboard, and IME behavior.
- Put a concise error next to the field and include how to fix it.
- On failed submission, focus the first invalid field and retain all entered values.
- Use both a visual state and text for invalid input.
- Announce asynchronous success, error, and loading state through the best mechanism supported by the exact engine and platform.

## Accessibility Baseline

- Visible focus with sufficient contrast.
- Complete keyboard and controller routes.
- Text and controls that remain usable at increased UI scale.
- Targets sized and spaced for the intended pointer or touch input.
- Labels for fields, icon-only controls, values, and units.
- No information conveyed by color, sound, motion, or position alone.
- Captions or text equivalents for meaningful audio where required.
- Reduced motion, camera shake, flashing, and transparency options where relevant.
- Remappable gameplay input and alternatives to rapid, simultaneous, or hold-only gestures where the design allows.
- Localized text, fallback fonts, bidirectional layout, and understandable errors.
- Timeouts that can be extended or disabled when they govern essential interaction.

Native accessibility and screen-reader APIs evolved during Godot 4.x. Inspect the exact class reference and platform support before using accessibility metadata or `DisplayServer` accessibility methods. Prefer built-in controls, but do not claim assistive-technology support until it has been tested in an exported build with the target tool.

## Composite and Custom Controls

If a custom component contains several controls, decide whether it exposes one focus stop or several. Document its keys, value changes, and exit behavior.

For a single custom focus target:

- Set an appropriate `focus_mode`.
- Draw a dedicated focus state.
- Support `ui_accept` and relevant directional actions.
- Expose a meaningful label, value, state, and action through APIs available in the target release.
- Preserve pointer and touch parity.

Avoid using raw keycodes when an Input Map action or `Shortcut` can express the command.

## Common Failures

- No focus until the user clicks.
- A hidden overlay still intercepts pointer input.
- A modal opens behind the current focus or lets focus escape to the covered screen.
- Enter submits twice because both a button signal and global input handler react.
- Escape closes the whole app while a `LineEdit` or popup should consume it first.
- A disabled button looks active, or a focused button looks only hovered.
- Error text appears visually but focus and assistive output never indicate it.
- An icon-only action has no label or discoverable explanation.
