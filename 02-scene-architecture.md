# 02 — Scene Architecture & Design Patterns

> How to structure your game like a professional. Scenes, nodes, and the patterns that make Godot sing.

---

## Table of Contents

- [Godot's Philosophy](#godots-philosophy)
- [The Node System](#the-node-system)
- [Scene Composition](#scene-composition)
- [Scene Inheritance](#scene-inheritance)
- [The Component Pattern](#the-component-pattern)
- [The State Machine Pattern](#the-state-machine-pattern)
- [The Observer Pattern (Signals)](#the-observer-pattern)
- [The Singleton Pattern (Autoloads)](#the-singleton-pattern)
- [The Command Pattern](#the-command-pattern)
- [Object Pooling](#object-pooling)
- [Dependency Injection in Godot](#dependency-injection)
- [Scene Tree Best Practices](#scene-tree-best-practices)

---

## Godot's Philosophy

Godot's architecture is fundamentally different from Unity or Unreal:

| Concept | Unity | Unreal | Godot |
|---------|-------|--------|-------|
| Base unit | GameObject | Actor | **Node** |
| Composition | Components on GO | Components on Actor | **Child Nodes** |
| Prefabs | Prefabs | Blueprints | **Scenes (.tscn)** |
| Globals | Static classes | GameInstance | **Autoloads** |
| Events | C# Events/Delegates | Delegates | **Signals** |

**The golden rule:** In Godot, **everything is a node** and **everything is a scene**. A scene is just a saved tree of nodes. Scenes can contain other scenes. This is composition.

---

## The Node System

### Common Node Types

```
Node (base — no transform, no rendering)
├── Node2D (2D transform: position, rotation, scale)
│   ├── Sprite2D
│   ├── CharacterBody2D
│   ├── RigidBody2D
│   ├── Area2D
│   ├── TileMapLayer
│   ├── Camera2D
│   └── AnimatedSprite2D
├── Node3D (3D transform)
│   ├── MeshInstance3D
│   ├── CharacterBody3D
│   ├── Camera3D
│   ├── DirectionalLight3D
│   └── WorldEnvironment
├── Control (UI node — anchors, themes)
│   ├── Label
│   ├── Button
│   ├── TextureRect
│   ├── VBoxContainer
│   └── Panel
├── AudioStreamPlayer
└── Timer
```

### When to Use Which Body Node

```
CharacterBody2D/3D
└── When YOU control movement (player characters, NPCs)
└── Has move_and_slide() — handles collisions for you
└── NOT affected by physics forces

RigidBody2D/3D
└── When PHYSICS controls movement (balls, crates, ragdolls)
└── Affected by gravity, forces, impulses
└── Don't set position directly! Use forces/impulses

StaticBody2D/3D
└── Doesn't move, but things collide with it (walls, floors)

Area2D/3D
└── Detects overlaps but doesn't physically collide
└── Perfect for: hitboxes, pickup zones, triggers
```

---

## Scene Composition

### The Right Way to Build a Player

Instead of one massive script, split into reusable scenes:

```
Player.tscn (CharacterBody2D)
├── Sprite2D
├── CollisionShape2D
├── AnimationPlayer
├── HitBox (Area2D)            ← separate scene
│   └── CollisionShape2D
├── HurtBox (Area2D)           ← separate scene
│   └── CollisionShape2D
├── HealthComponent (Node)     ← separate scene
├── StateMachine (Node)        ← separate scene
│   ├── IdleState
│   ├── WalkState
│   ├── JumpState
│   └── AttackState
└── Camera2D
```

### Reusable HitBox Scene

```gdscript
# hitbox.gd
extends Area2D
class_name HitBox

@export var damage: int = 10

func _ready() -> void:
    # HitBoxes detect HurtBoxes
    area_entered.connect(_on_area_entered)

func _on_area_entered(area: Area2D) -> void:
    if area is HurtBox:
        area.take_hit(self)
```

### Reusable HurtBox Scene

```gdscript
# hurtbox.gd
extends Area2D
class_name HurtBox

signal hit_received(hitbox: HitBox)

func take_hit(hitbox: HitBox) -> void:
    hit_received.emit(hitbox)
```

### Reusable Health Component

```gdscript
# health_component.gd
extends Node
class_name HealthComponent

signal health_changed(current: int, maximum: int)
signal died

@export var max_health: int = 100
var health: int

func _ready() -> void:
    health = max_health

func take_damage(amount: int) -> void:
    health = maxi(health - amount, 0)
    health_changed.emit(health, max_health)
    if health == 0:
        died.emit()

func heal(amount: int) -> void:
    health = mini(health + amount, max_health)
    health_changed.emit(health, max_health)
```

### Wiring It All Together

```gdscript
# player.gd
extends CharacterBody2D

@onready var health_comp: HealthComponent = $HealthComponent
@onready var hurtbox: HurtBox = $HurtBox

func _ready() -> void:
    hurtbox.hit_received.connect(_on_hit)
    health_comp.died.connect(_on_died)

func _on_hit(hitbox: HitBox) -> void:
    health_comp.take_damage(hitbox.damage)

func _on_died() -> void:
    queue_free()
```

---

## Scene Inheritance

Create a base scene then derive specialized versions:

### Base Enemy Scene (enemy_base.tscn)

```gdscript
# enemy_base.gd
extends CharacterBody2D
class_name EnemyBase

@export var speed: float = 50.0
@export var health: int = 30

func _physics_process(delta: float) -> void:
    _move(delta)
    move_and_slide()

# Virtual method — override in inheriting scenes
func _move(delta: float) -> void:
    pass

func take_damage(amount: int) -> void:
    health -= amount
    if health <= 0:
        _die()

func _die() -> void:
    queue_free()
```

### Derived: Slime (inherits enemy_base.tscn)

```gdscript
# slime.gd (extends enemy base scene)
extends EnemyBase

@export var bounce_height: float = -200.0
var bounce_timer: float = 0.0

func _move(delta: float) -> void:
    bounce_timer += delta
    # Bounce in a sine pattern
    velocity.x = sin(bounce_timer * 2.0) * speed
    velocity.y = bounce_height if is_on_floor() and bounce_timer > 1.0 else velocity.y + 980.0 * delta

func _die() -> void:
    # Custom death: split into smaller slimes
    spawn_mini_slimes(2)
    super._die()
```

### Derived: Flying Enemy

```gdscript
# flying_enemy.gd
extends EnemyBase

@export var hover_amplitude: float = 20.0

func _move(delta: float) -> void:
    var target_direction := global_position.direction_to(target.global_position)
    velocity = target_direction * speed
    # Add hover bobbing
    velocity.y += sin(Time.get_ticks_msec() * 0.003) * hover_amplitude
```

---

## The Component Pattern

The most powerful Godot pattern. Instead of inheritance, attach behavior through child nodes:

```gdscript
# velocity_component.gd
extends Node
class_name VelocityComponent

@export var max_speed: float = 200.0
@export var acceleration: float = 800.0
@export var friction: float = 600.0

var velocity: Vector2 = Vector2.ZERO

func accelerate_in_direction(direction: Vector2, delta: float) -> void:
    velocity = velocity.move_toward(direction * max_speed, acceleration * delta)

func decelerate(delta: float) -> void:
    velocity = velocity.move_toward(Vector2.ZERO, friction * delta)
    
func move(body: CharacterBody2D) -> void:
    body.velocity = velocity
    body.move_and_slide()
    velocity = body.velocity
```

```gdscript
# flash_component.gd — makes any sprite flash white when hit
extends Node
class_name FlashComponent

@export var sprite: Sprite2D
@export var flash_material: ShaderMaterial
var original_material: Material

func _ready() -> void:
    original_material = sprite.material

func flash() -> void:
    sprite.material = flash_material
    await get_tree().create_timer(0.1).timeout
    sprite.material = original_material
```

```gdscript
# knockback_component.gd
extends Node
class_name KnockbackComponent

var knockback_velocity: Vector2 = Vector2.ZERO
@export var friction: float = 600.0

func apply_knockback(direction: Vector2, force: float) -> void:
    knockback_velocity = direction * force

func update(delta: float) -> Vector2:
    knockback_velocity = knockback_velocity.move_toward(Vector2.ZERO, friction * delta)
    return knockback_velocity
```

Now any entity can mix and match:

```
Player.tscn
├── VelocityComponent
├── HealthComponent
├── FlashComponent
├── KnockbackComponent
└── ...

Goblin.tscn
├── VelocityComponent
├── HealthComponent
├── FlashComponent
├── KnockbackComponent
└── ...

Turret.tscn (stationary — no velocity/knockback)
├── HealthComponent
├── FlashComponent
└── ...
```

---

## The State Machine Pattern

Every game needs state machines. Here's the Godot-native way:

### State Base Class

```gdscript
# state.gd
extends Node
class_name State

# Reference to the state machine (set by StateMachine)
var state_machine: StateMachine

# Called when entering this state
func enter() -> void:
    pass

# Called when exiting this state
func exit() -> void:
    pass

# Called every frame
func process(delta: float) -> void:
    pass

# Called every physics frame
func physics_process(delta: float) -> void:
    pass

# Called on input
func handle_input(event: InputEvent) -> void:
    pass
```

### State Machine

```gdscript
# state_machine.gd
extends Node
class_name StateMachine

@export var initial_state: State
var current_state: State

func _ready() -> void:
    # Give each state a reference to this machine
    for child in get_children():
        if child is State:
            child.state_machine = self
    
    # Start initial state
    if initial_state:
        current_state = initial_state
        current_state.enter()

func _process(delta: float) -> void:
    if current_state:
        current_state.process(delta)

func _physics_process(delta: float) -> void:
    if current_state:
        current_state.physics_process(delta)

func _unhandled_input(event: InputEvent) -> void:
    if current_state:
        current_state.handle_input(event)

func transition_to(new_state_name: String) -> void:
    var new_state := get_node(new_state_name) as State
    if new_state == null:
        push_warning("State '%s' not found!" % new_state_name)
        return
    if new_state == current_state:
        return
    
    current_state.exit()
    current_state = new_state
    current_state.enter()
```

### Example States

```gdscript
# idle_state.gd
extends State

func enter() -> void:
    # Get the parent of the state machine (the player)
    var player := state_machine.get_parent() as Player
    player.anim_player.play("idle")

func physics_process(delta: float) -> void:
    var player := state_machine.get_parent() as Player
    var direction := Input.get_axis("move_left", "move_right")
    
    if direction != 0:
        state_machine.transition_to("WalkState")
    
    if Input.is_action_just_pressed("jump") and player.is_on_floor():
        state_machine.transition_to("JumpState")
```

```gdscript
# walk_state.gd
extends State

func enter() -> void:
    var player := state_machine.get_parent() as Player
    player.anim_player.play("walk")

func physics_process(delta: float) -> void:
    var player := state_machine.get_parent() as Player
    var direction := Input.get_axis("move_left", "move_right")
    
    if direction == 0:
        state_machine.transition_to("IdleState")
        return
    
    player.velocity.x = direction * player.speed
    player.move_and_slide()
    
    if Input.is_action_just_pressed("jump") and player.is_on_floor():
        state_machine.transition_to("JumpState")
```

---

## The Singleton Pattern

**Autoloads** are Godot's singletons — globally accessible scripts/scenes:

### Setting Up Autoloads

`Project → Project Settings → Globals/Autoload`

### Common Autoloads

```gdscript
# game_manager.gd (Autoload)
extends Node

signal game_paused
signal game_resumed
signal game_over

var score: int = 0
var is_paused: bool = false

func pause_game() -> void:
    is_paused = true
    get_tree().paused = true
    game_paused.emit()

func resume_game() -> void:
    is_paused = false
    get_tree().paused = false
    game_resumed.emit()

func add_score(points: int) -> void:
    score += points

func reset() -> void:
    score = 0
    is_paused = false
```

```gdscript
# scene_manager.gd (Autoload)
extends Node

signal scene_changed(scene_name: String)

var current_scene: Node

func _ready() -> void:
    current_scene = get_tree().current_scene

func change_scene(scene_path: String) -> void:
    # Deferred to avoid issues during physics
    call_deferred("_deferred_change_scene", scene_path)

func _deferred_change_scene(scene_path: String) -> void:
    current_scene.free()
    var new_scene := load(scene_path) as PackedScene
    current_scene = new_scene.instantiate()
    get_tree().root.add_child(current_scene)
    get_tree().current_scene = current_scene
    scene_changed.emit(scene_path)

# Transition with loading screen
func change_scene_with_transition(scene_path: String) -> void:
    # Fade out
    var tween := create_tween()
    var overlay := ColorRect.new()
    overlay.color = Color.BLACK
    overlay.color.a = 0.0
    overlay.set_anchors_preset(Control.PRESET_FULL_RECT)
    get_tree().root.add_child(overlay)
    
    tween.tween_property(overlay, "color:a", 1.0, 0.3)
    await tween.finished
    
    # Change scene
    _deferred_change_scene(scene_path)
    
    # Fade in
    tween = create_tween()
    tween.tween_property(overlay, "color:a", 0.0, 0.3)
    await tween.finished
    overlay.queue_free()
```

```gdscript
# event_bus.gd (Autoload) — Global signal hub
extends Node

# Define all global signals here
signal player_died
signal enemy_killed(enemy_type: String, position: Vector2)
signal item_collected(item_id: String)
signal checkpoint_reached(checkpoint_id: String)
signal dialog_started(dialog_id: String)
signal dialog_ended
signal achievement_unlocked(achievement_id: String)
```

From anywhere in your project:
```gdscript
# Emit
EventBus.enemy_killed.emit("goblin", global_position)

# Listen
EventBus.enemy_killed.connect(_on_enemy_killed)
```

---

## The Command Pattern

Useful for undo/redo systems, replays, and input buffering:

```gdscript
# command.gd
extends RefCounted
class_name Command

func execute() -> void:
    pass

func undo() -> void:
    pass
```

```gdscript
# move_command.gd
extends Command
class_name MoveCommand

var entity: Node2D
var direction: Vector2
var distance: float

func _init(p_entity: Node2D, p_direction: Vector2, p_distance: float) -> void:
    entity = p_entity
    direction = p_direction
    distance = p_distance

func execute() -> void:
    entity.position += direction * distance

func undo() -> void:
    entity.position -= direction * distance
```

```gdscript
# command_manager.gd
extends Node
class_name CommandManager

var history: Array[Command] = []
var redo_stack: Array[Command] = []

func execute(command: Command) -> void:
    command.execute()
    history.append(command)
    redo_stack.clear()

func undo() -> void:
    if history.is_empty():
        return
    var command := history.pop_back()
    command.undo()
    redo_stack.append(command)

func redo() -> void:
    if redo_stack.is_empty():
        return
    var command := redo_stack.pop_back()
    command.execute()
    history.append(command)
```

---

## Object Pooling

Critical for bullets, particles, and frequently spawned objects:

```gdscript
# object_pool.gd
extends Node
class_name ObjectPool

@export var scene: PackedScene
@export var initial_size: int = 20

var pool: Array[Node] = []

func _ready() -> void:
    # Pre-instantiate objects
    for i in initial_size:
        var obj := scene.instantiate()
        obj.set_process(false)
        obj.set_physics_process(false)
        obj.visible = false
        add_child(obj)
        pool.append(obj)

func get_object() -> Node:
    # Find an inactive object
    for obj in pool:
        if not obj.visible:
            obj.visible = true
            obj.set_process(true)
            obj.set_physics_process(true)
            return obj
    
    # Pool exhausted — grow it
    var obj := scene.instantiate()
    add_child(obj)
    pool.append(obj)
    return obj

func return_object(obj: Node) -> void:
    obj.visible = false
    obj.set_process(false)
    obj.set_physics_process(false)
```

Usage:
```gdscript
@onready var bullet_pool: ObjectPool = $BulletPool

func shoot() -> void:
    var bullet := bullet_pool.get_object()
    bullet.global_position = muzzle.global_position
    bullet.direction = aim_direction
    bullet.activate()
```

---

## Dependency Injection

Instead of hardcoding paths, inject dependencies through exports:

```gdscript
# ❌ BAD: Hardcoded path (breaks if you move the node)
func _ready() -> void:
    var player = get_node("../../Player")

# ❌ BAD: Using get_parent (fragile coupling)
func _ready() -> void:
    var player = get_parent().get_parent()

# ✅ GOOD: Export the dependency
@export var player: CharacterBody2D

# ✅ GOOD: Use groups
func _ready() -> void:
    var players := get_tree().get_nodes_in_group("player")
    if not players.is_empty():
        player = players[0]

# ✅ GOOD: Use signals (loosest coupling)
# Parent connects child signals in their own _ready()
```

---

## Scene Tree Best Practices

### Naming Conventions

```
PascalCase for node names:    Player, EnemySpawner, MainMenu
snake_case for files:         player.gd, enemy_spawner.tscn
UPPER_SNAKE for constants:    MAX_SPEED, TILE_SIZE
```

### Folder Structure

See [18 — Project Structure](./18-project-structure.md) for the complete guide.

### Scene Organization Rules

1. **Scenes should be self-contained** — don't rely on external nodes existing
2. **Signals flow UP** (child → parent), **method calls flow DOWN** (parent → child)
3. **Sibling nodes communicate through the parent** or through signals
4. **One script per node** (extend, don't attach helper scripts)
5. **Keep scenes small** — if it has more than ~15 nodes, split into sub-scenes

---

*← [01 — GDScript Mastery](./01-gdscript-mastery.md) | [03 — Physics & Collision](./03-physics-and-collision.md) →*
