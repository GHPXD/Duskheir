# Persistence Boundaries

## Persist Snapshots, Not the Scene Tree

A save file should contain plain, validated data that can rebuild runtime state. Suitable values include:

- Booleans, numbers, and strings.
- Arrays and dictionaries of other save-safe values.
- Stable content IDs.
- Grid coordinates or transforms represented as numeric fields.
- Random seeds and progression counters when deterministic restoration needs them.

Do not serialize live nodes, object instance IDs, callables, signal connections, or assumptions about one scene's child paths. Resource file paths are also fragile identities when content can move.

## Define a Snapshot Contract

Keep conversion near the state model but disk access in a storage boundary:

```gdscript
func to_snapshot() -> Dictionary:
	return {
		"coins": _coins,
		"current_map_id": String(_current_map_id),
		"unlocked_ids": _unlocked_ids.duplicate(),
	}

func restore_snapshot(snapshot: Dictionary) -> void:
	_coins = int(snapshot["coins"])
	_current_map_id = StringName(snapshot["current_map_id"])
	_unlocked_ids.assign(snapshot["unlocked_ids"])
```

Call `restore_snapshot` only with data that a separate validation step has accepted. Avoid defaulting silently inside every assignment because that can create a half-valid session.

## Version the Envelope

Wrap domain snapshots in an envelope:

```json
{
  "schema": 1,
  "run": {
    "coins": 12,
    "current_map_id": "harbor",
    "unlocked_ids": ["dash"]
  }
}
```

Increment the schema when interpretation changes, not for every added optional field. For each historical version that remains supported, provide a deterministic migration to the current shape. Otherwise reject it clearly and preserve the original file for recovery.

## Keep File I/O in One Boundary

A compact JSON store for non-sensitive game data can look like this:

```gdscript
class_name SaveStore
extends RefCounted

const CURRENT_SCHEMA: int = 1
const SAVE_PATH: String = "user://run.json"
const TEMP_PATH: String = "user://run.json.tmp"

func write_run(snapshot: Dictionary) -> Error:
	var envelope: Dictionary = {
		"schema": CURRENT_SCHEMA,
		"run": snapshot,
	}
	var serialized: String = JSON.stringify(envelope)
	var file := FileAccess.open(TEMP_PATH, FileAccess.WRITE)
	if file == null:
		return FileAccess.get_open_error()

	if not file.store_string(serialized):
		var write_error: Error = file.get_error()
		file.close()
		DirAccess.remove_absolute(TEMP_PATH)
		return write_error if write_error != OK else ERR_FILE_CANT_WRITE
	file.flush()
	var flush_error: Error = file.get_error()
	file.close()
	if flush_error != OK:
		DirAccess.remove_absolute(TEMP_PATH)
		return flush_error

	var replace_error: Error = DirAccess.rename_absolute(TEMP_PATH, SAVE_PATH)
	if replace_error != OK:
		DirAccess.remove_absolute(TEMP_PATH)
	return replace_error

func read_run() -> Dictionary:
	if not FileAccess.file_exists(SAVE_PATH):
		return {}

	var file := FileAccess.open(SAVE_PATH, FileAccess.READ)
	if file == null:
		push_warning("Save file could not be opened.")
		return {}

	var decoded: Variant = JSON.parse_string(file.get_as_text())
	if not decoded is Dictionary:
		push_warning("Save file is not a dictionary envelope.")
		return {}

	var envelope := decoded as Dictionary
	var schema_value: Variant = envelope.get("schema")
	if typeof(schema_value) != TYPE_INT and typeof(schema_value) != TYPE_FLOAT:
		push_warning("Save schema has the wrong type.")
		return {}
	var schema_number: float = float(schema_value)
	if not is_finite(schema_number) or schema_number != floorf(schema_number):
		push_warning("Save schema is not a finite integer.")
		return {}
	if int(schema_number) != CURRENT_SCHEMA:
		push_warning("Save schema is unsupported.")
		return {}

	var run_value: Variant = envelope.get("run", {})
	if run_value is Dictionary:
		return (run_value as Dictionary).duplicate(true)
	return {}
```

Adapt error reporting, backups, encoding, encryption, and replacement behavior to the project and target platforms. The temporary file prevents a failed serialization or write from truncating the current save; test whether replacement provides the durability guarantees the project needs. JSON is readable and portable but does not preserve every Godot type automatically. Convert vectors, colors, enums, and keyed dictionaries explicitly.

## Validate Before Restore

Validation should produce a clean current-version snapshot or an error, without touching live state.

Check:

- Required top-level keys and schema.
- Expected type of every value.
- Numeric ranges and finite floating-point values.
- Maximum collection and string sizes.
- Duplicate IDs and impossible combinations.
- Content IDs against the current catalog.
- Parent-child references and coordinate bounds.
- Whether missing content is fatal, dropped, or replaced with a fallback.

Treat user files as untrusted input. Do not load executable scripts or arbitrary resources named by save data.

## Plan Migrations

A migration pipeline should move one version at a time:

```text
v1 snapshot -> migrate_1_to_2 -> v2 snapshot -> validate current -> restore
```

Keep migrations pure where practical: input dictionary in, new dictionary out. Preserve fixtures for old versions so migrations remain testable after future changes.

When an authored asset is renamed, keep a stable ID or an explicit old-ID mapping. Do not infer identity from localized display text.

## Define Write Safety

For progress that matters, consider:

- Writing a temporary file before replacing the primary file.
- Keeping one known-good backup.
- Flushing and checking errors at the platform boundary.
- Serializing a complete snapshot before opening the destination for write.
- Preventing simultaneous writes.
- Saving at explicit safe points rather than every field mutation.

Use the exact `FileAccess` and `DirAccess` APIs available in the project's Godot version, and test replacement behavior on target platforms.

## Keep Save Domains Separate

Different lifetimes may deserve separate files or sections:

- Settings: device and accessibility preferences.
- Profile: unlocks, achievements, and meta-progression.
- Run/checkpoint: temporary game progress.
- Cache: disposable derived data.

Deleting a run should not erase settings. Resetting settings should not invalidate a profile. A cache failure should not block a save.

## Persistence Verification Matrix

- No file: start with documented defaults.
- Current valid file: exact round trip.
- Empty and truncated file: reject without partial restore.
- Wrong JSON types: reject or sanitize according to policy.
- Missing optional keys: apply documented defaults.
- Missing required keys: fail validation.
- Old supported schema: migrate and load.
- Future/unknown schema: preserve and reject safely.
- Unknown content ID: follow fallback policy.
- Save failure: keep live state and report a recoverable error.
- Two saves in sequence: newest complete state loads.
- New run: old checkpoint is ignored, replaced, or deleted according to policy.
- Exported build: `user://` resolves and permissions behave on every target platform.
