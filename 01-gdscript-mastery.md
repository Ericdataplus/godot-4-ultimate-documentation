# 01 — GDScript Mastery

> Everything you need to write clean, fast, maintainable GDScript. From basics to black-belt techniques.

---

## Table of Contents

- [The Basics Done Right](#the-basics-done-right)
- [Static Typing (Use It!)](#static-typing)
- [Variables & Constants](#variables--constants)
- [Functions & Return Types](#functions--return-types)
- [Annotations Deep Dive](#annotations-deep-dive)
- [Signals — The Backbone](#signals--the-backbone)
- [Lambda Functions](#lambda-functions)
- [Coroutines & Await](#coroutines--await)
- [Getters & Setters](#getters--setters)
- [Custom Resources](#custom-resources)
- [Enums & Constants Patterns](#enums--constants-patterns)
- [String Formatting](#string-formatting)
- [Array & Dictionary Tricks](#array--dictionary-tricks)
- [Pattern Matching](#pattern-matching)
- [Script Documentation](#script-documentation)
- [Performance Tips](#performance-tips)

---

## The Basics Done Right

Every GDScript file is implicitly a class. When you create a script and attach it to a node, that script **extends** the node's type.

```gdscript
# Every script starts with extends
extends CharacterBody2D

# Class name is optional but useful for type hints and the Create Node dialog
class_name Player
```

### Why `class_name` Matters

- Makes your class appear in the "Add Node" dialog
- Enables static typing: `var player: Player`
- Allows `is` checks: `if node is Player:`
- Required for custom resources

> **⚠️ Gotcha:** `class_name` must be globally unique across your entire project!

---

## Static Typing

**This is the single most impactful thing you can do for your code quality.** Static typing in GDScript gives you:

- Compile-time error catching
- Better autocompletion
- **Up to 100% performance improvement** for typed code in tight loops
- Self-documenting code

### Before (Untyped — Don't Do This)

```gdscript
var speed = 200
var health = 100
var direction = Vector2.ZERO
var enemy_list = []

func take_damage(amount):
    health -= amount
    if health <= 0:
        die()
```

### After (Typed — Do This!)

```gdscript
var speed: float = 200.0
var health: int = 100
var direction: Vector2 = Vector2.ZERO
var enemy_list: Array[Enemy] = []

func take_damage(amount: int) -> void:
    health -= amount
    if health <= 0:
        die()
```

### Type Inference

GDScript can infer types using `:=` — the variable gets the type of the right-hand side:

```gdscript
var speed := 200.0       # float (inferred)
var name := "Player"     # String (inferred)
var pos := Vector2.ZERO  # Vector2 (inferred)

# This WON'T work — type is locked
# speed = "fast"  # ERROR: Cannot assign String to float
```

### When to Use `:=` vs `: Type =`

```gdscript
# Use := when the type is obvious from the value
var timer := 0.0
var count := 0
var label := $Label as Label

# Use : Type = when you want to be explicit or the type differs
var speed: float = 200       # 200 is int, but we want float
var node: Node2D = null      # null has no type information
var items: Array[Item] = []  # typed arrays need explicit type
```

---

## Variables & Constants

### Variable Scopes

```gdscript
extends Node

# Class-level (accessible by all methods in this script)
var class_var: int = 0

# Constants — evaluated at compile time
const MAX_SPEED: float = 500.0
const TILE_SIZE: int = 16

# Static variables — shared across ALL instances of this class
static var instance_count: int = 0

# @onready — initialized when the node enters the scene tree
@onready var sprite: Sprite2D = $Sprite2D
@onready var collision: CollisionShape2D = $CollisionShape2D
@onready var anim_player: AnimationPlayer = $AnimationPlayer

func _ready() -> void:
    instance_count += 1

func _process(delta: float) -> void:
    # Local variable — only exists in this function
    var local_speed: float = MAX_SPEED * delta
```

### @onready vs _ready()

```gdscript
# These two are equivalent:

# Option 1: @onready (preferred — cleaner)
@onready var sprite: Sprite2D = $Sprite2D

# Option 2: _ready assignment
var sprite: Sprite2D
func _ready() -> void:
    sprite = $Sprite2D
```

> **💡 Pro Tip:** Hold `Ctrl` and drag a node from the Scene Tree into your script to auto-generate an `@onready` declaration!

---

## Functions & Return Types

### Always Declare Return Types

```gdscript
# Void functions (no return value)
func take_damage(amount: int) -> void:
    health -= amount

# Functions that return a value
func get_damage_multiplier() -> float:
    return 1.5 if is_critical else 1.0

# Functions that return complex types
func get_enemies_in_range(radius: float) -> Array[Enemy]:
    var result: Array[Enemy] = []
    # ... logic
    return result

# Optional return (returns null if condition not met)
func find_nearest_enemy() -> Enemy:
    if enemies.is_empty():
        return null
    # ... find logic
    return nearest
```

### Virtual Functions (Override These)

These are the "lifecycle" functions Godot calls automatically:

```gdscript
extends CharacterBody2D

# Called once when the node enters the scene tree
func _ready() -> void:
    pass

# Called every frame (for visual updates, input, non-physics logic)
# delta = time since last frame (typically ~0.016 at 60fps)
func _process(delta: float) -> void:
    pass

# Called at fixed intervals (for physics, movement)
# delta = fixed physics timestep (typically 0.016666 at 60 ticks/sec)
func _physics_process(delta: float) -> void:
    pass

# Called for every input event
func _input(event: InputEvent) -> void:
    pass

# Called for unhandled input (after UI and _input)
func _unhandled_input(event: InputEvent) -> void:
    pass

# Called when the node is about to be removed from the tree
func _exit_tree() -> void:
    pass

# Called when the node enters the tree (BEFORE _ready)
# Children may NOT be ready yet! Use for: registering, adding to groups
func _enter_tree() -> void:
    pass

# Called when the node receives a notification
func _notification(what: int) -> void:
    pass
```

> **⚠️ Common Mistake:** Don't put movement code in `_process()`. Use `_physics_process()` for anything involving `CharacterBody2D.move_and_slide()`, `RigidBody`, or physics queries.

---

## Annotations Deep Dive

### @export — Expose Variables to the Inspector

```gdscript
# Basic export
@export var speed: float = 200.0
@export var jump_force: float = -400.0

# Export with range slider
@export_range(0, 100, 1) var health: int = 100
@export_range(0.0, 1.0, 0.01) var friction: float = 0.5

# Export enums
@export_enum("Idle", "Walk", "Run", "Jump") var state: int = 0

# Export file paths
@export_file("*.tscn") var next_level: String
@export_dir var save_directory: String

# Export node paths (for referencing nodes without @onready)
@export var target_path: NodePath

# Export resources
@export var weapon_data: WeaponResource
@export var inventory: Array[ItemResource] = []

# Export flags (bitmask layers)
@export_flags("Fire", "Water", "Earth", "Wind") var elements: int = 0
@export_flags_2d_physics var collision_layer: int

# Export groups (organize Inspector)
@export_group("Movement")
@export var walk_speed: float = 100.0
@export var run_speed: float = 200.0

@export_group("Combat")
@export var attack_damage: int = 10
@export var attack_range: float = 50.0

# Export subgroups
@export_subgroup("Advanced")
@export var crit_chance: float = 0.1

# Export categories (bold header)
@export_category("Player Settings")
```

### @tool — Run Scripts in the Editor

```gdscript
@tool
extends Sprite2D

@export var radius: float = 100.0:
    set(value):
        radius = value
        queue_redraw()  # Redraw when changed in editor

func _draw() -> void:
    # This runs in the editor too!
    draw_circle(Vector2.ZERO, radius, Color.RED)

func _process(delta: float) -> void:
    if Engine.is_editor_hint():
        # Editor-only logic
        return
    # Game-only logic here
```

### @icon — Custom Node Icons

```gdscript
@icon("res://assets/icons/player.svg")
extends CharacterBody2D
class_name Player
```

---

## Signals — The Backbone

Signals are Godot's **Observer Pattern** implementation. They're the #1 way to communicate between nodes without tight coupling.

### Declaring Signals

```gdscript
# Simple signal (no data)
signal died
signal level_completed

# Signal with parameters
signal health_changed(new_health: int, max_health: int)
signal damage_taken(amount: int, source: Node)
signal item_collected(item: ItemResource, quantity: int)
```

### Emitting Signals

```gdscript
func take_damage(amount: int, attacker: Node) -> void:
    health -= amount
    health_changed.emit(health, max_health)
    damage_taken.emit(amount, attacker)
    
    if health <= 0:
        died.emit()
```

### Connecting Signals

```gdscript
# Method 1: Connect in code (preferred for complex logic)
func _ready() -> void:
    # Connect to own signal
    health_changed.connect(_on_health_changed)
    
    # Connect to child's signal
    $Button.pressed.connect(_on_button_pressed)
    
    # Connect to another node's signal
    var player := get_node("/root/Main/Player") as Player
    player.died.connect(_on_player_died)

func _on_health_changed(new_health: int, max_health: int) -> void:
    # Update UI
    health_bar.value = new_health

# Method 2: Connect with a lambda (great for one-liners)
func _ready() -> void:
    $Button.pressed.connect(func(): print("Button pressed!"))
    
    # Lambda with parameters
    $Player.health_changed.connect(
        func(hp: int, max_hp: int): health_label.text = str(hp)
    )

# Method 3: One-shot connection (auto-disconnects after first emit)
func _ready() -> void:
    $AnimationPlayer.animation_finished.connect(
        _on_intro_finished, CONNECT_ONE_SHOT
    )
```

### Signal Best Practices

```gdscript
# ✅ GOOD: Signal goes UP the tree (child emits, parent connects)
# Player.gd
signal died
# Main.gd
func _ready():
    $Player.died.connect(_on_player_died)

# ❌ BAD: Signal goes DOWN the tree (parent emits to child)
# Use direct method calls instead when going down
func _ready():
    $Player.take_damage(10)  # Just call the method directly

# ✅ GOOD: Use groups for broadcasting
func _ready():
    add_to_group("enemies")

# Somewhere else:
get_tree().call_group("enemies", "alert", player_position)
```

---

## Lambda Functions

Lambda functions (anonymous functions) are incredibly useful for short callbacks:

```gdscript
# Tween with lambda callback
func flash_red() -> void:
    var tween := create_tween()
    tween.tween_property(sprite, "modulate", Color.RED, 0.1)
    tween.tween_property(sprite, "modulate", Color.WHITE, 0.1)
    tween.finished.connect(func(): print("Flash done!"))

# Sorting with lambda
var scores: Array[int] = [45, 12, 89, 23, 67]
scores.sort_custom(func(a: int, b: int): return a > b)  # Descending

# Filtering
var enemies := get_tree().get_nodes_in_group("enemies")
var alive_enemies := enemies.filter(
    func(e: Node): return e.health > 0
)

# Mapping
var names := enemies.map(
    func(e: Node): return e.name
)

# Signal connection with deferred cleanup
var timer := get_tree().create_timer(2.0)
timer.timeout.connect(func():
    print("2 seconds passed!")
    queue_free()
)
```

---

## Coroutines & Await

The `await` keyword pauses a function until a signal is emitted:

```gdscript
# Wait for a timer
func spawn_with_delay() -> void:
    print("Spawning in 2 seconds...")
    await get_tree().create_timer(2.0).timeout
    print("Spawned!")
    spawn_enemy()

# Wait for an animation to finish
func play_death_animation() -> void:
    anim_player.play("death")
    await anim_player.animation_finished
    queue_free()

# Wait for a custom signal
func wait_for_player_input() -> void:
    show_dialog("Press any key to continue...")
    await input_received
    hide_dialog()

# Chain multiple awaits
func cutscene() -> void:
    await walk_to(Vector2(500, 300))
    await say("Hello there!")
    await get_tree().create_timer(1.0).timeout
    await say("Let's go!")
    await walk_to(Vector2(100, 300))

# ⚠️ DANGER: Don't await inside _process or _physics_process!
# It causes stacking execution contexts
func _process(delta: float) -> void:
    # ❌ NEVER DO THIS
    # await get_tree().create_timer(1.0).timeout
    pass
```

---

## Getters & Setters

Run custom logic when variables are read or written:

```gdscript
var health: int = 100:
    set(value):
        var old_health := health
        health = clampi(value, 0, max_health)
        health_changed.emit(health, max_health)
        if health <= 0 and old_health > 0:
            died.emit()
    get:
        return health

var max_health: int = 100

# Property with only a setter
var score: int = 0:
    set(value):
        score = value
        score_label.text = "Score: %d" % score

# Computed property (getter only, no backing variable needed)
var is_dead: bool:
    get: return health <= 0

var is_moving: bool:
    get: return velocity.length() > 0.1

# Percentage-based property
var health_percent: float:
    get: return float(health) / float(max_health)
```

---

## Custom Resources

Custom Resources are Godot's most **underrated** feature. Use them for data containers:

```gdscript
# weapon_resource.gd
extends Resource
class_name WeaponResource

@export var name: String = ""
@export var damage: int = 10
@export var attack_speed: float = 1.0
@export var range: float = 50.0
@export_multiline var description: String = ""
@export var icon: Texture2D
@export var sound_effect: AudioStream

func get_dps() -> float:
    return damage * attack_speed
```

Now you can create weapon data in the Inspector without any code:

1. Right-click in FileSystem → New Resource → WeaponResource
2. Fill in the properties
3. Use it anywhere:

```gdscript
# player.gd
@export var equipped_weapon: WeaponResource

func attack() -> void:
    var damage := equipped_weapon.damage
    # Apply damage...
```

### Nested Resources

```gdscript
# inventory_slot.gd
extends Resource
class_name InventorySlot

@export var item: ItemResource
@export var quantity: int = 1

# inventory.gd  
extends Resource
class_name Inventory

@export var slots: Array[InventorySlot] = []
@export var max_slots: int = 20

func add_item(item: ItemResource, amount: int = 1) -> bool:
    # Check existing stacks first
    for slot in slots:
        if slot.item == item:
            slot.quantity += amount
            return true
    
    # Add to new slot
    if slots.size() < max_slots:
        var new_slot := InventorySlot.new()
        new_slot.item = item
        new_slot.quantity = amount
        slots.append(new_slot)
        return true
    
    return false  # Inventory full
```

---

## Enums & Constants Patterns

```gdscript
# Simple enum
enum State { IDLE, WALKING, RUNNING, JUMPING, FALLING }
var current_state: State = State.IDLE

# Enum with explicit values (useful for serialization)
enum Direction {
    UP = 0,
    RIGHT = 1,
    DOWN = 2,
    LEFT = 3,
}

# Flag enum (bitmask)
enum DamageType {
    PHYSICAL = 1,
    FIRE = 2,
    ICE = 4,
    LIGHTNING = 8,
    POISON = 16,
}

# Check flags
var resistances: int = DamageType.FIRE | DamageType.ICE
func is_resistant_to(type: DamageType) -> bool:
    return (resistances & type) != 0

# Constants file (create as autoload for global access)
# constants.gd
class_name Constants

const GRAVITY: float = 980.0
const TILE_SIZE: int = 16
const MAX_ENEMIES: int = 50

# Layer names
const LAYER_PLAYER: int = 1
const LAYER_ENEMY: int = 2
const LAYER_PROJECTILE: int = 4
const LAYER_ENVIRONMENT: int = 8
```

---

## String Formatting

```gdscript
# Format operator (%)
var text := "HP: %d / %d" % [health, max_health]
# Result: "HP: 75 / 100"

# Common format specifiers
var examples := """
Integer:   %d      → %d
Float:     %f      → %f  
2 decimals: %.2f   → %.2f
String:    %s      → %s
Hex:       %x      → %x
Padded:    %05d    → %05d
""" % [42, 42, 3.14159, 3.14159, 3.14159, 3.14159, "hello", "hello", 255, 255, 42, 42]

# String interpolation (Godot 4.x — still limited, prefer %)
var player_name := "Hero"
var msg := "Hello " + player_name + "! You have " + str(health) + " HP"

# Multiline strings
var dialog := """
This is a multiline string.
It preserves line breaks.
Useful for long text or JSON templates.
"""

# String methods
var input := "  Hello World  "
input.strip_edges()       # "Hello World"
input.to_lower()          # "  hello world  "
input.to_upper()          # "  HELLO WORLD  "
input.split(" ")          # ["", "", "Hello", "World", "", ""]
"path/to/file.gd".get_file()         # "file.gd"
"path/to/file.gd".get_extension()    # "gd"
"path/to/file.gd".get_base_dir()     # "path/to"
```

---

## Array & Dictionary Tricks

### Arrays

```gdscript
# Typed arrays (always prefer these)
var scores: Array[int] = [10, 20, 30, 40, 50]
var enemies: Array[Enemy] = []

# Useful methods
scores.append(60)
scores.insert(0, 5)           # Insert at beginning
scores.pop_back()             # Remove & return last
scores.pop_front()            # Remove & return first
scores.erase(30)              # Remove first occurrence of 30
scores.has(20)                # true
scores.find(20)               # returns index, or -1
scores.size()                 # length
scores.is_empty()             # true if empty
scores.shuffle()              # randomize order
scores.reverse()              # reverse in place
scores.sort()                 # ascending sort
scores.pick_random()          # random element

# Functional programming
var doubled := scores.map(func(x: int): return x * 2)
var even := scores.filter(func(x: int): return x % 2 == 0)
var total := scores.reduce(func(acc: int, x: int): return acc + x, 0)
var has_high := scores.any(func(x: int): return x > 40)
var all_positive := scores.all(func(x: int): return x > 0)

# Slicing
var first_three := scores.slice(0, 3)
var last_two := scores.slice(-2)

# Array comprehension (using map/filter)
# Get positions of all enemies within range
var nearby_positions := enemies.filter(
    func(e: Enemy): return e.global_position.distance_to(position) < 200.0
).map(
    func(e: Enemy): return e.global_position
)
```

### Dictionaries

```gdscript
# Basic dictionary
var player_data: Dictionary = {
    "name": "Hero",
    "level": 5,
    "class": "Warrior",
    "stats": {
        "str": 15,
        "dex": 10,
        "int": 8,
    }
}

# Access
var name: String = player_data["name"]
var name2: String = player_data.get("name", "Unknown")  # With default

# Check existence
if player_data.has("name"):
    print(player_data["name"])

# Iteration
for key in player_data:
    print("%s: %s" % [key, player_data[key]])

for key in player_data.keys():
    pass

for value in player_data.values():
    pass

# Merging
var defaults := {"health": 100, "speed": 200.0, "name": "Unknown"}
var overrides := {"name": "Hero", "speed": 300.0}
defaults.merge(overrides, true)  # overwrite = true

# Using enum keys (great for stat systems)
enum Stat { STRENGTH, DEXTERITY, INTELLIGENCE, CONSTITUTION }
var stats: Dictionary = {
    Stat.STRENGTH: 10,
    Stat.DEXTERITY: 14,
    Stat.INTELLIGENCE: 8,
    Stat.CONSTITUTION: 12,
}
```

---

## Pattern Matching

GDScript has a `match` statement (similar to `switch` in other languages):

```gdscript
# Basic match
match current_state:
    State.IDLE:
        play_animation("idle")
    State.WALKING:
        play_animation("walk")
    State.RUNNING:
        play_animation("run")
    _:  # Default case (like 'else')
        play_animation("idle")

# Match with multiple values
match direction:
    Direction.UP, Direction.DOWN:
        print("Vertical")
    Direction.LEFT, Direction.RIGHT:
        print("Horizontal")

# Match with binding (capture the value)
match event:
    var e when e is InputEventKey:
        print("Key pressed: ", e.keycode)
    var e when e is InputEventMouseButton:
        print("Mouse button: ", e.button_index)

# Match with arrays
match command:
    ["move", var x, var y]:
        move_to(Vector2(x, y))
    ["attack", var target]:
        attack(target)
    ["heal", var amount]:
        heal(amount)
    _:
        print("Unknown command")

# Match with dictionaries
match data:
    {"type": "damage", "amount": var amt}:
        take_damage(amt)
    {"type": "heal", "amount": var amt}:
        heal(amt)
```

---

## Script Documentation

Document your code and it shows up in Godot's built-in help!

```gdscript
## A player character that handles movement, combat, and interaction.
##
## The Player class manages all player-related functionality including
## physics-based movement, health/damage systems, and NPC interaction.
## [br][br]
## Usage:
## [codeblock]
## var player = Player.new()
## player.speed = 200.0
## [/codeblock]
extends CharacterBody2D
class_name Player

## Emitted when the player takes damage.
## [param amount] The damage dealt.
## [param source] The node that dealt the damage.
signal damage_taken(amount: int, source: Node)

## Maximum movement speed in pixels per second.
@export var speed: float = 200.0

## Current health points. Setting this below 0 triggers death.
var health: int = 100

## Calculate the damage after applying armor reduction.
## [br][br]
## Returns the final damage amount after mitigation.
## Returns [code]0[/code] if the player is invincible.
func calculate_damage(raw_damage: int, damage_type: DamageType) -> int:
    if is_invincible:
        return 0
    return maxi(raw_damage - armor, 1)
```

---

## Performance Tips

### The Top 10 GDScript Performance Rules

```gdscript
# 1. CACHE NODE REFERENCES (don't call get_node every frame)
# ❌ Bad
func _process(delta: float) -> void:
    $Sprite2D.rotation += delta  # get_node called every frame!

# ✅ Good
@onready var sprite: Sprite2D = $Sprite2D
func _process(delta: float) -> void:
    sprite.rotation += delta

# 2. USE STATIC TYPING EVERYWHERE
# ❌ Bad
var speed = 200
func move(dir):
    position += dir * speed

# ✅ Good (compiles to faster bytecode)
var speed: float = 200.0
func move(dir: Vector2) -> void:
    position += dir * speed

# 3. USE distance_squared_to() FOR COMPARISONS
# ❌ Bad (sqrt is expensive)
if position.distance_to(target) < 100.0:
    attack()

# ✅ Good (compare squared distances)
if position.distance_squared_to(target) < 10000.0:  # 100^2
    attack()

# 4. PREFER SIGNALS OVER POLLING
# ❌ Bad (checking every frame)
func _process(delta: float) -> void:
    if health <= 0:
        die()

# ✅ Good (only triggers on change)
var health: int = 100:
    set(value):
        health = value
        if health <= 0:
            die()

# 5. PRE-CALCULATE CONSTANTS
# ❌ Bad
func _physics_process(delta: float) -> void:
    var angle := position.angle_to_point(target.position)
    var direction := Vector2(cos(angle), sin(angle))

# ✅ Good  
func _physics_process(delta: float) -> void:
    var direction := position.direction_to(target.position)  # Built-in!

# 6. USE OBJECT POOLING FOR FREQUENT SPAWN/DESPAWN
# See the Object Pooling pattern in 02-scene-architecture.md

# 7. MINIMIZE NODES IN THE SCENE TREE
# Nodes not in the tree don't process, but having too many is still overhead

# 8. USE ARRAYS OVER NODES FOR DATA
# ❌ Bad: 1000 Label nodes for inventory
# ✅ Good: Array of data, rendered dynamically

# 9. AVOID STRING OPERATIONS IN HOT PATHS
# String concatenation creates new objects — use StringName for comparisons

# 10. USE BUILT-IN METHODS OVER MANUAL IMPLEMENTATIONS
# Godot's built-in methods are implemented in C++ and are MUCH faster
# Examples: lerp(), clamp(), move_toward(), direction_to(), etc.
```

---

## Bonus: Ternary Expressions

```gdscript
# Single-line conditionals
var label_text: String = "Alive" if health > 0 else "Dead"
var speed_mult: float = 2.0 if is_sprinting else 1.0
var direction: int = -1 if facing_left else 1
```

---

*Next: [02 — Scene Architecture](./02-scene-architecture.md) →*

---

## Debugging Tricks

```gdscript
# Print the scene tree (invaluable for complex scenes)
print_tree_pretty()  # Indented tree output to console

# Find memory leaks — prints nodes that aren't in ANY tree
print_orphan_nodes()

# Assert for development (stripped from release builds)
assert(enemy != null, "Enemy reference was null!")
assert(health > 0, "Health was %d, expected positive" % health)
# Game CRASHES with your message — catches bugs immediately

# Duck typing — check if a method/property exists
if target.has_method("take_damage"):
    target.take_damage(10)
if target.has_signal("died"):
    target.died.connect(_on_target_died)
```

---

## GDScript Limitations to Know

```
Current GDScript (4.x) does NOT have:

❌ Generics — No Array<Enemy> syntax (use Array[Enemy] for typed arrays,
   but custom generic types aren't supported)
❌ Traits/Interfaces — Can't define abstract contracts
   (Workaround: duck typing with has_method(), or assert in _ready)
❌ Sum Types / Tuples — No Result<Ok, Err> pattern
❌ Multi-signal await — Can't await multiple signals at once
   (Workaround: create a helper that merges signals)
❌ Abstract classes — No way to force subclass to implement a method
   (Workaround: assert(false, "Override this!") in base class)

These are PLANNED for future versions.
```

---
