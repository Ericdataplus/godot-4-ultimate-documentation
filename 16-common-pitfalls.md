# 16 — Common Pitfalls & Gotchas

> Every "why isn't this working?!" moment, diagnosed and solved. Sourced from community pain.

---

## Table of Contents

- [The Classics](#the-classics)
- [Physics Gotchas](#physics-gotchas)
- [Signal Gotchas](#signal-gotchas)
- [Scripting Gotchas](#scripting-gotchas)
- [Resource Gotchas](#resource-gotchas)
- [UI Gotchas](#ui-gotchas)
- [Editor Gotchas](#editor-gotchas)
- [Architecture Gotchas](#architecture-gotchas)
- [Export Gotchas](#export-gotchas)

---

## The Classics

### 1. Node References are null in _ready()

```gdscript
# ❌ PROBLEM: @onready variables are null when accessed from another script's _ready
# The order scripts run _ready() depends on tree order (children first, then parent)

# ✅ FIX: Use call_deferred or await
func _ready() -> void:
    await get_tree().process_frame  # Wait one frame
    var player := get_tree().get_first_node_in_group("player")
    # Now it's guaranteed to be initialized
```

### 2. Movement in _process Instead of _physics_process

```gdscript
# ❌ PROBLEM: Inconsistent movement at different framerates
func _process(delta: float) -> void:
    velocity.x = 200
    move_and_slide()

# ✅ FIX: Physics movement goes in _physics_process
func _physics_process(delta: float) -> void:
    velocity.x = 200
    move_and_slide()

# Rule: move_and_slide(), move_and_collide(), any CharacterBody
# movement, raycasts → _physics_process
# Visual-only updates (UI, particles, camera) → _process
```

### 3. Diagonal Movement is Faster

```gdscript
# ❌ PROBLEM: Moving diagonally = √2 ≈ 1.414x faster
var direction := Vector2(
    Input.get_axis("left", "right"),
    Input.get_axis("up", "down")
)
velocity = direction * speed  # Diagonal = ~141% speed!

# ✅ FIX: Use Input.get_vector (auto-normalizes)
var direction := Input.get_vector("left", "right", "up", "down")
velocity = direction * speed
# Or manually: direction = direction.normalized()
```

---

## Physics Gotchas

### 4. Collision Shape Changed During Physics = Crash

```gdscript
# ❌ CRASH: Modifying collision shapes during physics callbacks
func _on_body_entered(body: Node2D) -> void:
    $CollisionShape2D.disabled = true  # BOOM!

# ✅ FIX: Always use set_deferred for physics object changes
func _on_body_entered(body: Node2D) -> void:
    $CollisionShape2D.set_deferred("disabled", true)
```

### 5. CharacterBody Bouncing on Slopes

```gdscript
# ❌ Character hops/bounces when walking down slopes

# ✅ FIX: Enable floor snapping
floor_snap_length = 16.0   # Must be long enough to reach slope surface
floor_stop_on_slope = true  # Prevents sliding down slopes when idle
floor_constant_speed = true # Consistent speed going up/down slopes
```

### 6. Physics Bodies Passing Through Each Other

```
Common causes:
1. Objects moving too fast (tunneling)
   → Enable CCD (Continuous Collision Detection) on the RigidBody
2. Collision shapes too thin
   → Make thicker, or use raycasts for fast projectiles
3. Physics tick rate too low + high speeds
   → Increase ticks/sec or reduce speeds
4. Collision layers/masks not overlapping
   → Check the layer matrix in Project Settings
5. Missing CollisionShape child
   → Every physics body NEEDS a CollisionShape child!
```

### 7. Scaling RigidBody Directly

```gdscript
# ❌ PROBLEM: Scaling a RigidBody resets to 1:1 at runtime
$RigidBody2D.scale = Vector2(2, 2)  # Gets overridden!

# ✅ FIX: Scale the CHILD nodes (sprite, collision shape) instead
# The CollisionShape2D has its own size properties
# Scale the visual (Sprite2D) and match the collision shape dimensions
$RigidBody2D/Sprite2D.scale = Vector2(2, 2)
$RigidBody2D/CollisionShape2D.shape.radius = 32.0  # Double the default
```

### 8. Scene Changes in Physics Callbacks

```gdscript
# ❌ ERROR: Changing scenes during physics callbacks
func _on_body_entered(body: Node2D) -> void:
    get_tree().change_scene_to_file("res://next.tscn")  # Error!

# ✅ FIX: Defer scene changes  
func _on_body_entered(body: Node2D) -> void:
    call_deferred("_change_scene")

func _change_scene() -> void:
    get_tree().change_scene_to_file("res://next.tscn")
```

---

## Signal Gotchas

### 9. Signals Not Firing

```gdscript
# Debug checklist:
# 1. Collision layers/masks set correctly?
print("Layer: ", collision_layer, " Mask: ", collision_mask)
# 2. "monitoring" and "monitorable" enabled on Area2D?
print("Monitoring: ", monitoring, " Monitorable: ", monitorable)
# 3. Signal actually connected? (check editor or code)
# 4. Node freed before signal emitted?
# 5. For body_entered: entering body MUST have a CollisionShape
# 6. For area_entered: BOTH areas need monitoring/monitorable

# Quick debug — does ANY input/event reach this node?
func _ready() -> void:
    for sig in get_signal_list():
        print("Signal: ", sig.name)
```

### 10. Connecting Same Signal Twice

```gdscript
# ❌ PROBLEM: Connecting a signal multiple times fires the callback multiple times
func _ready() -> void:
    button.pressed.connect(_on_pressed)
    # ... later, maybe in a function called again:
    button.pressed.connect(_on_pressed)  # Now fires TWICE!

# ✅ FIX: Use CONNECT_ONE_SHOT or check if connected
if not button.pressed.is_connected(_on_pressed):
    button.pressed.connect(_on_pressed)
# Or use connect flags:
button.pressed.connect(_on_pressed, CONNECT_ONE_SHOT)
```

---

## Scripting Gotchas

### 11. await in _process Creates Coroutine Stacking

```gdscript
# ❌ PROBLEM: Each frame starts a NEW coroutine that stacks up
func _process(delta: float) -> void:
    await get_tree().create_timer(1.0).timeout
    print("This runs more and more times!")

# ✅ FIX: Use a flag or timer node instead
var _can_act := true
func _process(delta: float) -> void:
    if not _can_act:
        return
    _can_act = false
    do_something()
    await get_tree().create_timer(1.0).timeout
    _can_act = true
```

### 12. await After Node is Freed

```gdscript
# ❌ CRASH: The node is freed while a coroutine is waiting
func attack() -> void:
    play_animation("attack")
    await $AnimationPlayer.animation_finished
    # If this node is queue_free'd during the animation ^
    # this line crashes because "self" no longer exists!
    apply_damage()

# ✅ FIX: Check if still valid, or cancel on tree_exiting
func attack() -> void:
    play_animation("attack")
    await $AnimationPlayer.animation_finished
    if not is_instance_valid(self):
        return
    apply_damage()
```

### 13. Tween Not Working / Old Tween Still Playing

```gdscript
# ❌ PROBLEM: Creating new tweens without killing old ones
func flash() -> void:
    var tween := create_tween()  # Old tween still running!
    tween.tween_property(self, "modulate", Color.RED, 0.1)

# ✅ FIX: Kill the previous tween
var _current_tween: Tween
func flash() -> void:
    if _current_tween and _current_tween.is_valid():
        _current_tween.kill()
    _current_tween = create_tween()
    _current_tween.tween_property(self, "modulate", Color.RED, 0.1)
```

### 14. Empty _process() Still Costs Performance

```gdscript
# ❌ PROBLEM: Having an empty _process in your script
func _process(delta: float) -> void:
    pass  # This still gets called every frame, for every instance!

# ✅ FIX: Don't include it at all, or disable it
# Simply remove the function if you don't need per-frame updates
# If you sometimes need it:
func _ready() -> void:
    set_process(false)  # Off by default
```

### 15. get_node Returns null for Unique Names

```gdscript
# ❌ Forgetting the % prefix for unique names
var label := get_node("HealthLabel")  # null if it's a unique name!

# ✅ Use % for scene-unique names
var label := %HealthLabel

# ⚠️ SCOPE WARNING: Unique names are scoped to their OWN scene
# You can't use %NodeName to find a node inside an instanced subscene
# From a parent scene, do: $InstancedScene.get_node("%InternalNode")
```

---

## Resource Gotchas

### 16. Shared Resources Affecting All Instances

```gdscript
# ❌ PROBLEM: Changing a resource on one instance changes ALL instances
# Resources are shared by default when duplicating scenes/nodes

# ✅ FIX: Make the resource unique
func _ready() -> void:
    material = material.duplicate()  # Now independent
    
# Or in the editor: right-click resource → "Make Unique"

# This affects: Materials, ShaderMaterials, StyleBoxes, Fonts,
# Shapes (CollisionShape resources), and any custom Resource
```

### 17. Export Variables Reset After Changing Script

```
After changing @export variables in a script, already-placed nodes
keep their OLD values. They don't pick up new defaults.

FIX: 
1. Select the node in the editor
2. Right-click the property → "Reset to default"
Or delete and re-add the node

Also: Renaming an @export variable in script makes the editor
lose the stored value (it's a new property name).
```

---

## UI Gotchas

### 18. Input Not Detected (UI Eating Events)

```gdscript
# Common causes:
# 1. Input action not defined in Project Settings → Input Map
# 2. Using _input instead of _unhandled_input (UI eating events!)
# 3. A Control node is intercepting mouse events
# 4. Node is paused but process_mode isn't PROCESS_MODE_ALWAYS
# 5. Node is not in the scene tree (not added or freed)

# Debug which Control ate the click:
# Debugger → Misc tab → shows which Control was clicked

# FIX: Use _unhandled_input for gameplay controls
func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("attack"):
        attack()

# FIX: Set mouse_filter on non-interactive UI elements
$BackgroundPanel.mouse_filter = Control.MOUSE_FILTER_IGNORE
```

### 19. UI Anchors Not Working

```
❌ Element jumps to wrong position on different screen sizes

Common causes:
1. Anchors set but rect_position manually adjusted afterward
   → Keep rect_position at (0,0), use margins for offset
2. Anchors only work when parent is a Control or Viewport
   → Don't put Control nodes under Node2D
3. Not using "Full Rect" preset for root UI node
   → Select root UI node → Layout → Full Rect
4. Containers override manual positioning of children
   → Children of containers can't be manually positioned
```

### 20. Sprite Looks Blurry (Pixel Art)

```
FOR PIXEL ART:
Project Settings → Rendering → Textures → Default Texture Filter → "Nearest"

OR per-texture:
Import dock → Filter → change to "Nearest"
Click "Re-import"

Project Settings → Display → Window:
  Stretch Mode: "viewport" (integer scaling)
  Stretch Aspect: "keep"
```

---

## Editor Gotchas

### 21. Autoload Order Matters

```
Autoloads initialize TOP to BOTTOM in Project Settings
If AutoloadB depends on AutoloadA, A must be listed FIRST
Otherwise B's _ready() runs before A exists

# Check current order:
# Project Settings → Autoload → drag to reorder
```

### 22. Timer Doesn't Fire When Paused

```gdscript
# ❌ Timer stops during tree pause
get_tree().paused = true
# All timers with default process_mode stop!

# ✅ FIX: Set process mode on the timer
$Timer.process_mode = Node.PROCESS_MODE_ALWAYS

# For SceneTree timers:
get_tree().create_timer(1.0, true, false, true)  # Last arg = process_always
```

### 23. Particles Disappear Immediately

```gdscript
# ❌ One-shot particles queue_free too soon
func spawn_effect(pos: Vector2) -> void:
    var particles := particles_scene.instantiate()
    add_child(particles)
    particles.global_position = pos
    particles.emitting = true
    particles.queue_free()  # Freed before particles render!

# ✅ FIX: Wait for particles to finish
func spawn_effect(pos: Vector2) -> void:
    var particles := particles_scene.instantiate()
    particles.one_shot = true
    add_child(particles)
    particles.global_position = pos
    particles.emitting = true
    await particles.finished
    particles.queue_free()

# OR: Use particle preprocess to start particles "already spawned"
particles.preprocess = 1.0  # Simulate 1 second of emission instantly
```

---

## Architecture Gotchas

### 24. Depending on Scene Tree Layout

```gdscript
# ❌ FRAGILE: Code breaks when scene tree changes
var health_bar = get_node("../../UI/HUD/HealthBar")
var player = get_parent().get_parent().get_node("Player")

# ✅ FIX: Use groups, signals, @export, or Autoloads
# Groups:
var player := get_tree().get_first_node_in_group("player")

# @export:
@export var health_bar: ProgressBar  # Set in editor

# Autoload:
GameState.player_health  # Global access
```

### 25. Circular Dependencies

```
❌ Script A uses class_name B, Script B uses class_name A
   → Godot can't figure out load order → errors

FIX:
1. Refactor so dependencies flow one direction
2. Use signals instead of direct references
3. Use a shared Autoload as intermediary
4. Use duck typing: if node.has_method("damage"): node.damage(10)
```

### 26. Singleton (Autoload) Overuse

```
❌ Putting ALL game logic in Autoloads
   → Everything becomes global → spaghetti → untestable

✅ Rules for Autoloads:
   USE for: EventBus, AudioManager, SaveManager, SceneManager
   DON'T USE for: Player logic, enemy behavior, UI state
   
   If you have more than 5-6 Autoloads, you're probably
   putting too much logic in globals.
```

---

## Export Gotchas

### 27. Release Build Removes assert() and Debug Code

```gdscript
# assert() is stripped from RELEASE builds
assert(enemy != null)  # Silently does nothing in release!

# If you need runtime validation in release:
if enemy == null:
    push_error("Enemy is null!")
    return
```

### 28. Custom Resources Not Found After Export

```
❌ Custom Resource classes not included in export

FIX: Ensure every custom Resource class has a class_name
     and is referenced somewhere in the project
     (directly or via @export)

Also: Check export filters aren't excluding .tres files
```

---

*← [15 — Editor Secrets](./15-editor-secrets.md) | [17 — Tips & Tricks](./17-tips-and-tricks.md) →*
