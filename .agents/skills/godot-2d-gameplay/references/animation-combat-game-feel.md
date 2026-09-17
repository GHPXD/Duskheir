# Animation, Combat, and Game Feel

## Resolve Animation From State

Physics and combat decide state; animation displays it. Compute one animation
choice after authoritative movement and change clips only when the choice
changes.

```gdscript
extends AnimatedSprite2D

enum Locomotion { IDLE, RUN, RISE, FALL }

var _state: Locomotion = Locomotion.IDLE

func sync_from_body(body: CharacterBody2D) -> void:
    var next: Locomotion
    if not body.is_on_floor():
        next = Locomotion.RISE if body.velocity.y < 0.0 else Locomotion.FALL
    elif absf(body.velocity.x) > 1.0:
        next = Locomotion.RUN
    else:
        next = Locomotion.IDLE

    if next == _state:
        return
    _state = next
    play(_clip_for(_state))

func _clip_for(value: Locomotion) -> StringName:
    match value:
        Locomotion.RUN:
            return &"run"
        Locomotion.RISE:
            return &"rise"
        Locomotion.FALL:
            return &"fall"
        _:
            return &"idle"
```

Action states such as attack, hurt, stun, or death usually need priority over
locomotion. Represent that priority explicitly, and decide whether an animation
event or gameplay timer ends the action. Avoid mixing both without a single
authority.

Use `AnimationPlayer` for coordinated property, audio, method, and effect
tracks. Avoid keying the root physics position unless the animation is the
intentional movement authority. `AnimationTree` is appropriate once transitions
and blends outgrow a small state resolver.

## Health Is an Authoritative Component

Clamp inputs and emit one death event. Feedback systems subscribe; they do not
change health themselves.

```gdscript
class_name HealthComponent
extends Node

signal changed(current: int, maximum: int)
signal damaged(amount: int, remaining: int)
signal died

@export_range(1, 100000, 1) var maximum: int = 10
var current: int

func _ready() -> void:
    current = maximum
    changed.emit(current, maximum)

func apply_damage(requested: int) -> bool:
    if requested <= 0 or current <= 0:
        return false
    var applied: int = mini(requested, current)
    current -= applied
    damaged.emit(applied, current)
    changed.emit(current, maximum)
    if current == 0:
        died.emit()
    return true

func heal(requested: int) -> int:
    if requested <= 0 or current <= 0:
        return 0
    var before: int = current
    current = mini(current + requested, maximum)
    if current != before:
        changed.emit(current, maximum)
    return current - before
```

Decide whether healing can revive, whether armor modifies requested or applied
damage, and whether death removes, disables, or pools the actor. Encode those
rules rather than letting callers infer them.

## Hitbox and Hurtbox Contract

Area-to-area combat keeps combat overlap separate from body blocking. Put
hitboxes and hurtboxes on distinct named layers and connect `area_entered`.

```gdscript
class_name Hurtbox2D
extends Area2D

signal hit_received(amount: int, source_position: Vector2)

@export var health: HealthComponent
@export var team_id: int = 0

func receive_hit(amount: int, source_position: Vector2) -> bool:
    if health == null or not health.apply_damage(amount):
        return false
    hit_received.emit(amount, source_position)
    return true
```

```gdscript
class_name Hitbox2D
extends Area2D

@export_range(1, 10000, 1) var damage: int = 1
@export var team_id: int = 0
var _victims: Dictionary = {}

func begin_attack() -> void:
    _victims.clear()
    monitoring = true

func end_attack() -> void:
    monitoring = false

func _on_area_entered(area: Area2D) -> void:
    if not area is Hurtbox2D:
        return
    var hurtbox: Hurtbox2D = area as Hurtbox2D
    if hurtbox.team_id == team_id or _victims.has(hurtbox):
        return
    _victims[hurtbox] = true
    hurtbox.receive_hit(damage, global_position)
```

Connect the signal in the scene or `_ready()`. Clear the victim set once per
attack activation, not every frame. For invulnerability, put the rule on the
receiver so every damage source respects it.

## Projectiles and Reuse

Use an `Area2D` when overlap sampling at physics ticks is sufficient. Use a
`CharacterBody2D`, ray, or shape query when fast motion needs a swept test. Add
projectiles to a stable world-owned container so shooter transforms do not drag
them afterward.

A reusable projectile needs an explicit lifecycle:

```gdscript
extends Area2D

signal released(projectile: Area2D)

@onready var lifetime: Timer = %Lifetime
var direction: Vector2 = Vector2.RIGHT
var speed: float = 0.0
var active: bool = false

func _ready() -> void:
    visible = false
    monitoring = false
    monitorable = false
    set_physics_process(false)
    lifetime.stop()

func activate(origin: Vector2, travel_direction: Vector2, travel_speed: float) -> void:
    global_position = origin
    direction = travel_direction.normalized()
    speed = travel_speed
    active = true
    visible = true
    monitoring = true
    monitorable = true
    set_physics_process(true)
    lifetime.start()

func deactivate() -> void:
    if not active:
        return
    active = false
    visible = false
    set_physics_process(false)
    lifetime.stop()
    call_deferred(&"_finish_deactivate")

func _finish_deactivate() -> void:
    monitoring = false
    monitorable = false
    released.emit(self)

func _physics_process(delta: float) -> void:
    global_position += direction * speed * delta
```

Reset every mutable field that can leak between uses: transform, direction,
velocity, team, damage, timers, particles, animation, collision exceptions,
victim sets, and queued callbacks. Keep free and in-use collections in the pool
manager instead of treating `visible == false` as proof of availability.
Emit the pool release only after deferred collision shutdown, so the pool cannot
reactivate an instance before an older deferred property change lands.

## Feedback Stack

Attach feedback to confirmed events in this order:

1. Immediate state readability: pose, tint, or hit flash.
2. Spatial confirmation: particles, decal, knockback, or recoil.
3. Audio confirmation with controlled polyphony.
4. Camera response proportional to event importance.
5. Optional time emphasis for rare, high-value impacts.

Keep each layer independently tunable. Respect accessibility settings for
shake, flashes, and vibration. Do not shake UI unless explicitly intended.

A camera shake should add a temporary offset and always converge to zero:

```gdscript
extends Camera2D

@export_range(0.0, 100.0, 0.1) var max_offset: float = 8.0
@export_range(0.0, 10.0, 0.1) var decay: float = 2.5
var _trauma: float = 0.0
var _rng: RandomNumberGenerator = RandomNumberGenerator.new()

func add_trauma(amount: float) -> void:
    _trauma = clampf(_trauma + amount, 0.0, 1.0)

func _process(delta: float) -> void:
    _trauma = move_toward(_trauma, 0.0, decay * delta)
    var strength: float = _trauma * _trauma * max_offset
    if is_zero_approx(strength):
        offset = Vector2.ZERO
        return
    offset = Vector2(
        _rng.randf_range(-strength, strength),
        _rng.randf_range(-strength, strength)
    )
```

Use a dedicated visual RNG; never consume the gameplay generation RNG for
camera noise. If using global time scale for hit stop, restore it through one
owner even when the actor dies or the scene changes.

## Combat Tests

- A hit from the same team causes no health change.
- One swing overlapping for several frames damages once unless designed as a
  damage-over-time zone.
- Two hits in one physics tick clamp at zero and emit death once.
- A disabled or pooled hurtbox cannot receive a late overlap callback.
- A projectile expires, collides, and is reused with a fresh timer and victim
  state.
- Pausing the tree produces the intended cooldown and invulnerability behavior.
- Disabling shake leaves camera offset exactly zero.
