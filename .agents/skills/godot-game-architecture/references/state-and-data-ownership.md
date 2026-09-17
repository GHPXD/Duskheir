# State and Data Ownership

## Build an Ownership Record

For each affected value, answer:

- What does it mean in game terms?
- Who is allowed to change it?
- Who reads it?
- When is it created?
- When is it reset?
- Must it survive a scene replacement, a run, or an application restart?
- Is it authored definition data, mutable runtime state, presentation state, or a saved snapshot?
- What stable identity represents it outside the live scene tree?

If two systems claim to own a value, resolve that conflict before adding synchronization code.

## Use the Narrowest Stable Owner

Examples:

- Current hit points: the actor's health component.
- Whether one door is unlocked: the room or door runtime model.
- Units remaining in one battle: the match root or match referee.
- Currency carried across several levels in one run: the run/session model.
- Permanently unlocked characters: the profile model.
- Master volume: settings service.
- Label text: the label, derived from authoritative state.

The narrowest owner minimizes reset work and prevents unrelated scenes from depending on state they do not need.

## Encapsulate Mutation

A runtime model can be independent of the scene tree:

```gdscript
class_name RunWallet
extends RefCounted

signal balance_changed(total: int)

var _balance: int = 0

func balance() -> int:
	return _balance

func reset(starting_balance: int = 0) -> void:
	_set_balance(maxi(0, starting_balance))

func credit(amount: int) -> void:
	if amount > 0:
		_set_balance(_balance + amount)

func try_spend(amount: int) -> bool:
	if amount < 0 or amount > _balance:
		return false
	_set_balance(_balance - amount)
	return true

func _set_balance(value: int) -> void:
	if value == _balance:
		return
	_balance = value
	balance_changed.emit(_balance)
```

Only this model writes `_balance`. Shops request `try_spend`; rewards request `credit`; UI reads `balance()` and observes `balance_changed`.

The owner of `RunWallet` determines its lifetime. A gameplay root can own it for one match, while a session autoload can own it across several level replacements.

## Initialize Views Reliably

Events do not retain history. A view should pull, then observe:

```gdscript
var _wallet: RunWallet

func bind(wallet: RunWallet) -> void:
	_wallet = wallet
	_update_balance(wallet.balance())
	wallet.balance_changed.connect(_update_balance)
```

If rebinding is supported, disconnect the old model first. Define whether the view can exist without a model and make that state visible during development.

## Keep Definition Resources Read-Mostly

Definition resources are well suited to authored data:

```gdscript
class_name ItemDefinition
extends Resource

@export var id: StringName
@export var display_name: String
@export_range(0, 999999, 1) var base_price: int
@export var icon: Texture2D
```

Inventory runtime state should store stable IDs and quantities, not mutate `base_price` or add an `owned_count` field to the shared definition.

Benefits of the split:

- One definition can serve many runtime instances.
- Balancing data remains editable without carrying save progress.
- Saves use compact stable IDs.
- Runtime reset does not alter imported or authored assets.
- Missing content can be handled during ID resolution.

If runtime modifiers are needed, store them in an instance model that references the definition ID.

## Separate Simulation and Presentation

Simulation owns facts and rules. Presentation derives what the player sees.

Examples:

- Store `current_health`; derive health-bar fill.
- Store selected unit ID; derive highlight visibility.
- Store crop growth progress; derive sprite frame.
- Store current objective state; derive localized text.

Do not save a label's formatted text when the underlying state can reproduce it. Do not make an animation callback the only record that a rule completed.

## Manager Responsibilities

A manager is useful when it coordinates peers at one lifetime boundary. Give it a domain-specific responsibility and API:

- Match referee: registers units and determines match completion.
- Room director: owns room encounters and door rules.
- Session model: owns run state across level scenes.
- Profile store: loads and saves account progress.

Avoid names and APIs that imply unlimited scope. If `GameManager` handles economy, audio, navigation, saves, scene changes, and UI paths, it has several owners hidden inside one global node.

## Reset Semantics

Define explicit operations rather than relying on construction timing:

- `start_new_run(seed)` creates clean run state.
- `start_match(config)` creates scene-local match state.
- `restore(snapshot)` replaces clean state from validated data.
- `return_to_menu()` states whether a run is abandoned or retained.
- `retry()` states whether random seed, inventory, and checkpoint state reset.

Call reset operations from one flow coordinator. Scattered resets create order-dependent bugs.

## Ownership Verification

- Search for direct assignments to the authoritative field; only its owner should remain.
- Create a late UI listener and confirm it initializes correctly.
- Start two runs in one process and compare initial state.
- Free a gameplay scene and confirm the session model contains no scene-node references.
- Edit a definition resource at runtime in a test and ensure the architecture prevents or intentionally isolates it.
- Trace one mutation from request through validation, state change, notification, and presentation.
