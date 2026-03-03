# 17 — Tips & Tricks

> 50+ pro tips that will change how you use Godot. Community-sourced, battle-tested.

---

## Quick Wins

### 1. Ternary Expressions Everywhere
```gdscript
var label := "Alive" if health > 0 else "Dead"
var flip := -1 if facing_left else 1
sprite.flip_h = velocity.x < 0 if velocity.x != 0 else sprite.flip_h
```

### 2. Input.get_vector() Normalizes Automatically
```gdscript
# Don't do this manually:
var dir := Vector2(Input.get_axis("left","right"), Input.get_axis("up","down")).normalized()

# Just do:
var dir := Input.get_vector("left", "right", "up", "down")
```

### 3. move_toward() is Your Best Friend
```gdscript
# Approach a value at fixed speed (NOT exponential like lerp)
health = move_toward(health, max_health, heal_rate * delta)
position.x = move_toward(position.x, target_x, speed * delta)
rotation = move_toward(rotation, target_angle, turn_speed * delta)
```

### 4. Math in Inspector Fields
```
Type "64*4" → Enter → 256
Type "360/8" → Enter → 45
Type "16+32" → Enter → 48
Works in any numeric field in the Inspector!
```

### 5. Use Groups for Bulk Operations
```gdscript
# Add to group
add_to_group("damageable")

# Call method on all group members
get_tree().call_group("damageable", "take_damage", 10)

# Get all members
var targets := get_tree().get_nodes_in_group("damageable")

# Set property on all
get_tree().set_group("collectible", "visible", true)

# Call deferred (safe for physics)
get_tree().call_group_flags(SceneTree.GROUP_CALL_DEFERRED, "enemy", "despawn")
```

### 6. One-Line Timer
```gdscript
await get_tree().create_timer(0.5).timeout
# Replaces: creating Timer node, connecting signal, etc.
```

### 7. Clamp Everything
```gdscript
health = clampi(health, 0, max_health)
position.x = clampf(position.x, 0.0, screen_width)
volume = clampf(volume, 0.0, 1.0)
# There's also wrapi/wrapf for looping values (angles, indices)
var angle := wrapf(angle, 0.0, TAU)
```

### 8. Random Utilities
```gdscript
randi() % 10              # Random int 0-9
randf()                    # Random float 0.0-1.0  
randf_range(-5.0, 5.0)    # Random float in range
randi_range(1, 6)          # Dice roll
[1, 2, 3].pick_random()   # Random array element

# Weighted random
func weighted_random(weights: Array[float]) -> int:
    var total := 0.0
    for w in weights:
        total += w
    var roll := randf() * total
    for i in weights.size():
        roll -= weights[i]
        if roll <= 0:
            return i
    return weights.size() - 1
```

---

## Hidden Editor Features

### 9. Color Picker in Code
```gdscript
# Right-click on Color() in the script editor → color picker appears!
# Click the colored box next to any Color property in Inspector too
var flash_color := Color(1, 0, 0)  # Right-click this → visual picker
```

### 10. Documentation Comments (##)
```gdscript
## This appears in Godot's built-in help system.
## You can Ctrl+Click a class_name in these comments to jump to it.
## Useful for documenting which class emits/receives signals.
##
## See also: [HealthComponent], [DamageSystem]
extends Node2D
class_name Player

## Maximum movement speed in pixels per second.
@export var speed: float = 200.0

## Emitted when health reaches zero.
signal died
```

### 11. Ctrl+Drag = Auto-complete Paths
```
Ctrl+Drag file from FileSystem into script → preload("res://path")
Ctrl+Drag node from SceneTree into script → @onready var node = $Path
Drag signal from editor to script → generates callback function
```

### 12. F6 Runs Current Scene (Not Main Scene)
```
F5 → Run project (main scene)
F6 → Run CURRENT scene (the one you're editing)
F7 → Debug the previously run scene
F8 → Stop running

This is essential for testing components in isolation!
Design every scene to be runnable on its own with F6.
```

### 13. Change Node Type Without Recreating
```
Right-click any node in SceneTree → "Change Type"
Converts a Node2D to Sprite2D, or Area2D to CharacterBody2D, etc.
Keeps the node name, position, and children!
```

### 14. Print the Scene Tree
```gdscript
# Dump the entire scene tree to console (great for debugging)
print_tree()         # Current node's subtree
print_tree_pretty()  # With indentation

# Also useful:
print_orphan_nodes()  # Find memory leaks — shows nodes not in tree
```

### 15. Debugger's Misc Tab
```
If UI clicks aren't registering:
1. Run the game
2. Open Debugger → Misc tab
3. Click in the game
4. The Misc tab shows WHICH Control node received the click
5. Find the invisible Control that's stealing your input!
```

### 16. Default Import Presets
```
Import dock → Preset dropdown → "Save Current as Default"

Now all future imports of that file type use your settings!
Great for: pixel art (Nearest filter), textures (compression), audio
```

---

## Scripting Tricks

### 17. owner Property
```gdscript
# 'owner' refers to the ROOT NODE of the packed scene this node belongs to
# Useful for finding the "entity" a component belongs to
func get_entity() -> Node:
    return owner  # Returns the root of the scene that owns this node
```

### 18. Type Checking and Safe Casts
```gdscript
if node is Player:
    node.take_damage(10)

if not enemy is Boss:
    enemy.stagger()

# Safe cast — returns null if wrong type
var player := node as Player
if player:
    player.heal(20)

# Duck typing — check if method exists
if target.has_method("take_damage"):
    target.take_damage(10)
```

### 19. assert() for Development
```gdscript
func set_weapon(weapon: WeaponResource) -> void:
    assert(weapon != null, "Weapon cannot be null!")
    assert(weapon.damage > 0, "Weapon damage must be positive!")
    equipped_weapon = weapon
# assert() only runs in debug builds — zero cost in release!
# Game CRASHES immediately with your message if assertion fails
# This is GOOD — it catches bugs early instead of hiding them
```

### 20. @tool for Editor Preview
```gdscript
@tool
extends Sprite2D
@export var color: Color = Color.WHITE:
    set(v):
        color = v
        modulate = v
# Now you see color changes live in the editor!
```

### 21. Export as Tool Button
```gdscript
@tool
extends Node

@export var regenerate: bool = false:
    set(value):
        if value and Engine.is_editor_hint():
            _regenerate_level()
        regenerate = false  # Reset so it can be "pressed" again

func _regenerate_level() -> void:
    # Procedural generation runs in editor!
    pass
```

### 22. Custom Editor Warnings
```gdscript
@tool
extends Node

func _get_configuration_warnings() -> PackedStringArray:
    var warnings: PackedStringArray = []
    if get_children().is_empty():
        warnings.append("This node needs at least one child!")
    return warnings
# Shows yellow ⚠️ warning triangles in the scene tree
```

### 23. Bitwise Flags for Compact State
```gdscript
enum StatusEffect {
    NONE    = 0,
    POISON  = 1,
    BURN    = 2,
    FREEZE  = 4,
    STUN    = 8,
    BLESS   = 16,
}

var _status: int = StatusEffect.NONE

func add_status(flag: StatusEffect) -> void: _status |= flag
func remove_status(flag: StatusEffect) -> void: _status &= ~flag
func has_status(flag: StatusEffect) -> bool: return (_status & flag) != 0

# Apply multiple: _status = StatusEffect.POISON | StatusEffect.BURN
```

### 24. Coroutine Chain Pattern
```gdscript
func play_sequence() -> void:
    await walk_to(target_position)
    await face_direction(Vector2.RIGHT)
    await show_dialog("Hello there!")
    await get_tree().create_timer(0.5).timeout
    await show_dialog("Take this item.")
    await give_item("key")
```

### 25. Signal-Based State Awaiting
```gdscript
# Wait for a specific event to occur
func wait_for_trigger() -> void:
    if not $Area2D.get_overlapping_bodies().is_empty():
        return  # Already triggered
    await $Area2D.body_entered
    print("Something entered the trigger!")
```

### 26. Functional Array Operations
```gdscript
var nodes := get_tree().get_nodes_in_group("target")

# Filter
var alive := nodes.filter(func(n): return n.health > 0)

# Find nearest
var nearest := nodes.reduce(func(best, n):
    if best == null: return n
    return n if n.global_position.distance_squared_to(pos) < best.global_position.distance_squared_to(pos) else best
)

# Sum values
var total := nodes.reduce(func(sum, n): return sum + n.score_value, 0)

# Check any/all
var any_alive := nodes.any(func(n): return n.health > 0)
var all_dead := nodes.all(func(n): return n.health <= 0)

# Map
var names := nodes.map(func(n): return n.name)
```

---

## Vector & Math Tricks

### 27. Useful Vector Operations
```gdscript
var dir := a.direction_to(b)           # Unit direction from A to B
var dist := a.distance_to(b)           # Distance (avoid in loops — use squared)
var angle := a.angle_to(b)             # Angle between vectors
var rotated := dir.rotated(PI / 4)     # Rotate vector
var perp := Vector2(-dir.y, dir.x)     # Perpendicular (2D)
var reflected := vel.bounce(normal)     # Reflect off surface
var projected := vel.project(dir)       # Project onto direction
```

### 28. Frame-Independent Lerp
```gdscript
# ❌ Frame-dependent (different at 30fps vs 144fps)
position = position.lerp(target, 0.1)

# ✅ Frame-independent (same at any framerate)
position = position.lerp(target, 1.0 - exp(-speed * delta))
# speed = higher is faster. 5.0-15.0 are common values.
# This is the correct formula. Everyone should use it.
```

### 29. Smooth Camera Follow
```gdscript
extends Camera2D

@export var target: Node2D
@export var smoothing: float = 10.0

func _physics_process(delta: float) -> void:
    if target:
        global_position = global_position.lerp(
            target.global_position,
            1.0 - exp(-smoothing * delta)
        )
```

---

## Game Feel Tricks

### 30. Coyote Time (Community Standard)
```gdscript
# Allow jumping for a short time after walking off a ledge
# Makes movement feel responsive and forgiving

@onready var coyote_timer: Timer = $CoyoteTimer  # One-shot, 0.15s

var _was_on_floor: bool = false

func _physics_process(delta: float) -> void:
    var on_floor := is_on_floor()
    
    # Just left the ground (but didn't jump)
    if _was_on_floor and not on_floor and velocity.y >= 0:
        coyote_timer.start()
    
    _was_on_floor = on_floor
    
    # Allow jump if on floor OR coyote time is active
    var can_jump := on_floor or not coyote_timer.is_stopped()
    
    if Input.is_action_just_pressed("jump") and can_jump:
        velocity.y = jump_force
        coyote_timer.stop()
```

### 31. Jump Buffering (Community Standard)
```gdscript
# Register jump presses slightly before landing
# Player presses jump 0.1s before touching ground → still jumps

@onready var buffer_timer: Timer = $JumpBufferTimer  # One-shot, 0.1s

func _physics_process(delta: float) -> void:
    if Input.is_action_just_pressed("jump"):
        if is_on_floor():
            velocity.y = jump_force
        else:
            buffer_timer.start()  # Buffer the input
    
    # Check buffer when landing
    if is_on_floor() and not buffer_timer.is_stopped():
        velocity.y = jump_force
        buffer_timer.stop()
```

### 32. Knockback That Feels Good
```gdscript
func apply_knockback(from_position: Vector2, force: float) -> void:
    var direction := global_position.direction_to(from_position) * -1
    velocity = direction * force

# Optional: hit stop for impact feeling
func hit_stop(duration: float = 0.05) -> void:
    Engine.time_scale = 0.02
    await get_tree().create_timer(duration * 0.02).timeout
    Engine.time_scale = 1.0
```

### 33. Screen Shake
```gdscript
# Add to your Camera script
func shake(intensity: float = 5.0, duration: float = 0.2) -> void:
    var tween := create_tween()
    for i in int(duration / 0.02):
        tween.tween_property(self, "offset",
            Vector2(randf_range(-1, 1), randf_range(-1, 1)) * intensity,
            0.02)
    tween.tween_property(self, "offset", Vector2.ZERO, 0.05)
```

### 34. Color Modulation for Feedback
```gdscript
# Flash on damage
func damage_flash() -> void:
    modulate = Color.RED
    var tween := create_tween()
    tween.tween_property(self, "modulate", Color.WHITE, 0.2)

# Blink during invincibility
func invincibility_blink(times: int = 5) -> void:
    var tween := create_tween().set_loops(times)
    tween.tween_property(self, "modulate:a", 0.3, 0.1)
    tween.tween_property(self, "modulate:a", 1.0, 0.1)
```

### 35. Trail Effect
```gdscript
func spawn_trail() -> void:
    var ghost := Sprite2D.new()
    ghost.texture = sprite.texture
    ghost.global_position = global_position
    ghost.modulate = Color(0.5, 0.5, 1.0, 0.5)
    get_parent().add_child(ghost)
    
    var tween := ghost.create_tween()
    tween.tween_property(ghost, "modulate:a", 0.0, 0.3)
    tween.tween_callback(ghost.queue_free)
```

---

## Visual Polish

### 36. The Juice Checklist
```
Every significant interaction should have (pick 3+):
1. Sound effect
2. Screen shake (subtle: 2-4px, heavy: 8-16px)
3. Particle effect
4. Squash & stretch on sprites
5. Hit stop / freeze frame (0.03-0.05s)
6. Camera zoom punch
7. UI feedback (flash, number popup)
8. Color flash (white or red)

These tiny details separate "feels janky" from "feels polished"
```

### 37. Particle Preprocess
```gdscript
# Particles "pop" into existence when scene loads
# Fix: preprocess simulates N seconds instantly
$GPUParticles2D.preprocess = 2.0
# Particles appear as if they've been running for 2 seconds
# Great for ambient effects (dust, rain, fireflies)
```

### 38. Screen Wrap
```gdscript
func _physics_process(delta: float) -> void:
    var screen := get_viewport_rect().size
    position.x = wrapf(position.x, 0, screen.x)
    position.y = wrapf(position.y, 0, screen.y)
```

---

## Architecture Tips

### 39. EventBus for Global Communication
See [02 — Scene Architecture](./02-scene-architecture.md) and [19 — Modular Architecture](./19-modular-architecture.md) for the full pattern.

### 40. Use Resources for Data, Not Nodes
```gdscript
# ❌ 100 enemy "data nodes" in the scene tree
# ✅ Array of Resource files with all stats
# See 19 — Modular Architecture for the data-driven pattern
```

### 41. Node Removed from Tree = Paused Processing
```gdscript
# Removing a node from the tree stops its _process/_physics_process
# Useful for "disabling" without freeing
remove_child(enemy)   # Processing stops, but enemy still exists!
add_child(enemy)      # Processing resumes

# ⚠️ Store a reference! Removed nodes are NOT freed but also NOT
# accessible via get_node anymore
```

### 42. Nodes Exist Before _ready
```gdscript
# Nodes exist in _enter_tree but aren't fully initialized
# Use _enter_tree for: registering with systems, adding to groups
# Use _ready for: everything else (children are guaranteed ready)

func _enter_tree() -> void:
    add_to_group("saveable")  # ✅ Good — lightweight setup

func _ready() -> void:
    $HealthComponent.max_health = data.health  # ✅ Good — children exist
```

---

*← [16 — Common Pitfalls](./16-common-pitfalls.md) | [18 — Project Structure](./18-project-structure.md) →*
