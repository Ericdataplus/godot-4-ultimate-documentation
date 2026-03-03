# 03 — Physics & Collision

> Master Godot's physics system. CharacterBody, RigidBody, collision layers, raycasting, and every gotcha explained.

---

## Table of Contents

- [Physics Bodies Overview](#physics-bodies-overview)
- [CharacterBody2D — Full Guide](#characterbody2d)
- [CharacterBody3D](#characterbody3d)
- [RigidBody2D/3D](#rigidbody)
- [StaticBody & AnimatableBody](#staticbody--animatablebody)
- [Area2D/3D — Detection Zones](#area2d3d)
- [Collision Layers & Masks](#collision-layers--masks)
- [Collision Shapes](#collision-shapes)
- [Raycasting](#raycasting)
- [ShapeCast & Physics Queries](#shapecast--physics-queries)
- [One-Way Platforms](#one-way-platforms)
- [Moving Platforms](#moving-platforms)
- [Alternative Physics Engines](#alternative-physics-engines)
- [Physics Performance Tips](#physics-performance-tips)
- [Physics Gotchas](#physics-gotchas)

---

## Physics Bodies Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    PHYSICS BODIES                           │
├──────────────────┬──────────────────┬───────────────────────┤
│  StaticBody2D    │ CharacterBody2D  │    RigidBody2D        │
│                  │                  │                       │
│  Doesn't move    │  YOU control it  │  PHYSICS controls it  │
│  Walls, floors   │  Players, NPCs   │  Balls, crates        │
│  Infinite mass   │  move_and_slide  │  Forces & impulses    │
│  Collides        │  Collides        │  Collides             │
│                  │                  │                       │
│  Also:           │  Full control    │  Don't set position!  │
│  AnimatableBody  │  over velocity   │  Use apply_force()    │
└──────────────────┴──────────────────┴───────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Area2D / Area3D                        │
│                                                             │
│  Detects overlaps — does NOT physically block anything      │
│  Perfect for: hitboxes, triggers, pickups, damage zones     │
│  Signals: body_entered, body_exited, area_entered           │
└─────────────────────────────────────────────────────────────┘
```

---

## CharacterBody2D

The workhorse for player characters and NPCs. You directly control the `velocity` vector, then call `move_and_slide()`.

### Complete 2D Platformer Controller

```gdscript
extends CharacterBody2D
class_name PlatformerPlayer

# Movement
@export var speed: float = 200.0
@export var acceleration: float = 1200.0
@export var friction: float = 1000.0

# Jump
@export var jump_velocity: float = -350.0
@export var gravity_scale: float = 1.0
@export var fall_gravity_multiplier: float = 1.5
@export var max_fall_speed: float = 600.0

# Jump tuning
@export var coyote_time: float = 0.1
@export var jump_buffer_time: float = 0.1
@export var variable_jump_cut: float = 0.4  # Release to cut jump short

var coyote_timer: float = 0.0
var jump_buffer_timer: float = 0.0
var was_on_floor: bool = false

# Get gravity from project settings (consistent with RigidBodies)
var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")

func _physics_process(delta: float) -> void:
    _apply_gravity(delta)
    _handle_jump()
    _handle_movement(delta)
    
    # Track floor state for coyote time
    was_on_floor = is_on_floor()
    
    move_and_slide()

func _apply_gravity(delta: float) -> void:
    if not is_on_floor():
        var gravity_mult := gravity_scale
        if velocity.y > 0:
            gravity_mult *= fall_gravity_multiplier
        velocity.y = minf(velocity.y + gravity * gravity_mult * delta, max_fall_speed)
    
    # Coyote time
    if was_on_floor and not is_on_floor() and velocity.y >= 0:
        coyote_timer = coyote_time
    else:
        coyote_timer = maxf(coyote_timer - delta, 0.0)

func _handle_jump() -> void:
    # Buffer jump input
    if Input.is_action_just_pressed("jump"):
        jump_buffer_timer = jump_buffer_time
    else:
        jump_buffer_timer = maxf(jump_buffer_timer - get_physics_process_delta_time(), 0.0)
    
    # Execute jump
    var can_jump := is_on_floor() or coyote_timer > 0.0
    if jump_buffer_timer > 0.0 and can_jump:
        velocity.y = jump_velocity
        jump_buffer_timer = 0.0
        coyote_timer = 0.0
    
    # Variable jump height (release early = shorter jump)
    if Input.is_action_just_released("jump") and velocity.y < 0:
        velocity.y *= variable_jump_cut

func _handle_movement(delta: float) -> void:
    var direction := Input.get_axis("move_left", "move_right")
    
    if direction != 0:
        velocity.x = move_toward(velocity.x, direction * speed, acceleration * delta)
    else:
        velocity.x = move_toward(velocity.x, 0.0, friction * delta)
```

### Complete Top-Down Controller

```gdscript
extends CharacterBody2D
class_name TopDownPlayer

@export var speed: float = 150.0
@export var acceleration: float = 800.0
@export var friction: float = 600.0

@onready var sprite: AnimatedSprite2D = $AnimatedSprite2D

func _physics_process(delta: float) -> void:
    var input_direction := Input.get_vector(
        "move_left", "move_right", "move_up", "move_down"
    )
    
    if input_direction != Vector2.ZERO:
        # Normalize to prevent diagonal speed boost
        input_direction = input_direction.normalized()
        velocity = velocity.move_toward(input_direction * speed, acceleration * delta)
        _update_animation(input_direction)
    else:
        velocity = velocity.move_toward(Vector2.ZERO, friction * delta)
        _play_idle()
    
    move_and_slide()

func _update_animation(direction: Vector2) -> void:
    if abs(direction.x) > abs(direction.y):
        sprite.play("walk_side")
        sprite.flip_h = direction.x < 0
    elif direction.y < 0:
        sprite.play("walk_up")
    else:
        sprite.play("walk_down")

func _play_idle() -> void:
    var anim := sprite.animation
    if anim == "walk_side":
        sprite.play("idle_side")
    elif anim == "walk_up":
        sprite.play("idle_up")
    else:
        sprite.play("idle_down")
```

### move_and_slide() Deep Dive

```gdscript
# move_and_slide() does a LOT under the hood:
# 1. Moves the body by velocity * delta
# 2. Detects collisions
# 3. Slides along surfaces it collides with
# 4. Updates internal collision data

# After calling move_and_slide(), you can query:
move_and_slide()

# Was there a collision?
if get_slide_collision_count() > 0:
    var collision := get_slide_collision(0)
    var collider := collision.get_collider()       # What we hit
    var normal := collision.get_normal()            # Surface normal
    var position := collision.get_position()        # Where we hit
    var remainder := collision.get_remainder()      # Remaining motion
    var travel := collision.get_travel()            # Distance traveled

# Floor/Wall/Ceiling detection (set by move_and_slide)
is_on_floor()    # Standing on something
is_on_wall()     # Touching a wall
is_on_ceiling()  # Hit the ceiling

# Get the floor normal (useful for slopes)
var floor_normal := get_floor_normal()  # Vector2.UP on flat ground

# Get what you're standing on
var floor_collider := get_last_slide_collision()
```

### Important CharacterBody2D Properties

```gdscript
# In the Inspector or in code:

# Motion Mode
motion_mode = MotionMode.GROUNDED  # Use for platformers (has floor/wall detection)
motion_mode = MotionMode.FLOATING  # Use for top-down (no floor/wall concept)

# Floor settings (CRITICAL for good movement feel)
floor_max_angle = deg_to_rad(45)    # Max slope angle counted as "floor"
floor_snap_length = 8.0             # Snap to floor — MUST SET for slopes!
                                    # Set high enough to reach floor on steepest slopes
floor_stop_on_slope = true          # Don't slide down slopes when idle
floor_constant_speed = true         # ⭐ Consistent speed going up/down slopes
                                    # Without this, character speeds up/slows down on inclines
floor_block_on_wall = true          # Prevent sliding on walls

# Wall settings
wall_min_slide_angle = deg_to_rad(15) # Minimum angle for wall sliding

# Max slides per frame (increase if getting stuck in geometry)
max_slides = 6
```

---

## CharacterBody3D

Same concepts, but in 3D:

```gdscript
extends CharacterBody3D
class_name Player3D

@export var speed: float = 5.0
@export var jump_velocity: float = 4.5
@export var mouse_sensitivity: float = 0.002

var gravity: float = ProjectSettings.get_setting("physics/3d/default_gravity")

@onready var camera_pivot: Node3D = $CameraPivot
@onready var camera: Camera3D = $CameraPivot/Camera3D

func _ready() -> void:
    Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseMotion:
        rotate_y(-event.relative.x * mouse_sensitivity)
        camera_pivot.rotate_x(-event.relative.y * mouse_sensitivity)
        camera_pivot.rotation.x = clampf(
            camera_pivot.rotation.x, deg_to_rad(-90), deg_to_rad(90)
        )

func _physics_process(delta: float) -> void:
    # Gravity
    if not is_on_floor():
        velocity.y -= gravity * delta
    
    # Jump
    if Input.is_action_just_pressed("jump") and is_on_floor():
        velocity.y = jump_velocity
    
    # Movement
    var input_dir := Input.get_vector("move_left", "move_right", "move_up", "move_down")
    var direction := (transform.basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()
    
    if direction:
        velocity.x = direction.x * speed
        velocity.z = direction.z * speed
    else:
        velocity.x = move_toward(velocity.x, 0, speed)
        velocity.z = move_toward(velocity.z, 0, speed)
    
    move_and_slide()
```

---

## RigidBody

Let the physics engine handle movement. **Never set position directly!**

```gdscript
extends RigidBody2D

# ✅ Correct: Use forces and impulses
func _ready() -> void:
    # Continuous force (like a thruster)
    apply_force(Vector2.UP * 500)
    
    # One-time impulse (like a jump or explosion)
    apply_impulse(Vector2(-100, -300))
    
    # Central impulse (no rotation)
    apply_central_impulse(Vector2.UP * 400)
    
    # Torque (rotation force)
    apply_torque(100.0)

# ❌ WRONG: Don't set position on RigidBody
# self.position = Vector2(100, 100)  # Physics engine fights this!

# If you MUST teleport a RigidBody:
func teleport(new_pos: Vector2) -> void:
    # Use PhysicsServer directly
    PhysicsServer2D.body_set_state(
        get_rid(),
        PhysicsServer2D.BODY_STATE_TRANSFORM,
        Transform2D(0, new_pos)
    )

# Freeze/Unfreeze
func freeze_in_place() -> void:
    freeze = true
    freeze_mode = RigidBody2D.FREEZE_MODE_STATIC  # Acts like StaticBody
    # or
    freeze_mode = RigidBody2D.FREEZE_MODE_KINEMATIC  # Acts like AnimatableBody

# Custom integrator for special physics
func _integrate_forces(state: PhysicsDirectBodyState2D) -> void:
    # Custom gravity
    var custom_gravity := Vector2(0, 500)
    state.apply_central_force(custom_gravity * mass)
    
    # Limit velocity
    if state.linear_velocity.length() > 500:
        state.linear_velocity = state.linear_velocity.normalized() * 500
```

### RigidBody Interaction Tips

```gdscript
# Push RigidBodies with CharacterBody
func _physics_process(delta: float) -> void:
    move_and_slide()
    
    for i in get_slide_collision_count():
        var collision := get_slide_collision(i)
        var collider := collision.get_collider()
        if collider is RigidBody2D:
            var push_direction := collision.get_normal() * -1
            collider.apply_central_impulse(push_direction * 80)
```

---

## Area2D/3D

Detects overlaps without blocking movement. Essential for game logic.

```gdscript
# pickup.gd
extends Area2D
class_name Pickup

signal collected(item_data: ItemResource)
@export var item: ItemResource

func _ready() -> void:
    body_entered.connect(_on_body_entered)

func _on_body_entered(body: Node2D) -> void:
    if body is Player:
        collected.emit(item)
        queue_free()
```

### Area2D Signals

```gdscript
# body_entered — when a PhysicsBody enters
# body_exited — when a PhysicsBody exits
# area_entered — when another Area enters
# area_exited — when another Area exits

func _ready() -> void:
    body_entered.connect(_on_body_entered)
    body_exited.connect(_on_body_exited)
    area_entered.connect(_on_area_entered)

# Monitoring vs Monitorable
# monitoring = true  → This area DETECTS others
# monitorable = true → This area CAN BE detected by others
```

### Damage Zone Example

```gdscript
# damage_zone.gd
extends Area2D

@export var damage_per_second: float = 10.0
var bodies_in_zone: Array[Node2D] = []

func _ready() -> void:
    body_entered.connect(func(body): bodies_in_zone.append(body))
    body_exited.connect(func(body): bodies_in_zone.erase(body))

func _physics_process(delta: float) -> void:
    for body in bodies_in_zone:
        if body.has_method("take_damage"):
            body.take_damage(damage_per_second * delta)
```

---

## Collision Layers & Masks

**The most confusing part of Godot physics** — but once it clicks, it's brilliant.

```
LAYER = "I exist on these layers" (what I AM)
MASK  = "I can see these layers" (what I DETECT/COLLIDE WITH)

Rule: Collision happens when A's MASK includes B's LAYER (or vice versa)
```

### Recommended Layer Setup

| Layer | Name          | Used By                           |
|-------|---------------|-----------------------------------|
| 1     | Player        | Player body                       |
| 2     | Enemy         | Enemy bodies                      |
| 3     | Environment   | Walls, floors, platforms          |
| 4     | Projectile    | Bullets, arrows                   |
| 5     | Pickup        | Items, coins                      |
| 6     | PlayerHitbox  | Player's attack hitbox            |
| 7     | EnemyHitbox   | Enemy attack hitboxes             |
| 8     | PlayerHurtbox | Player's hurtbox                  |
| 9     | EnemyHurtbox  | Enemy hurtboxes                   |
| 10    | Trigger       | Doors, spawn triggers, checkpoints|

### Setting Names

`Project Settings → General → Layer Names → 2D Physics`

### Configuration Examples

```
Player Body:
  Layer: 1 (Player)
  Mask:  3 (Environment) — collides with walls

Enemy Body:
  Layer: 2 (Enemy)
  Mask:  2, 3 (Enemy, Environment) — collides with other enemies and walls

Player Hitbox (sword swing):
  Layer: 6 (PlayerHitbox)
  Mask:  9 (EnemyHurtbox) — detects enemy hurtboxes only

Enemy Hurtbox:
  Layer: 9 (EnemyHurtbox)
  Mask:  6 (PlayerHitbox) — detected by player hitboxes only

Bullet:
  Layer: 4 (Projectile)
  Mask:  2, 3 (Enemy, Environment) — hits enemies and walls
```

### Setting Layers in Code

```gdscript
# Set specific layer/mask bits
collision_layer = 0           # Clear all layers
set_collision_layer_value(1, true)   # Enable layer 1

collision_mask = 0            # Clear all masks
set_collision_mask_value(3, true)    # Enable mask 3

# Bitmask math (if you know the layer numbers)
collision_layer = 1 << 0  # Layer 1 (bit 0)
collision_mask = (1 << 2) | (1 << 3)  # Layers 3 and 4
```

---

## Collision Shapes

### 2D Shapes (Performance: Best → Worst)

```
1. CircleShape2D        — Fastest. Use when possible.
2. RectangleShape2D     — Almost as fast. Great for boxes.
3. CapsuleShape2D       — Good for characters.
4. SegmentShape2D       — Lines. Good for one-way platforms.
5. ConvexPolygonShape2D — Custom shape but NO concavity.
6. ConcavePolygonShape2D — Any shape. SLOW. Use sparingly.
```

### Tips

```gdscript
# Disable collision temporarily
collision_shape.set_deferred("disabled", true)
# ⚠️ ALWAYS use set_deferred for collision changes during physics!
# Direct setting causes crashes!

# Multiple collision shapes (compound collider)
# Just add multiple CollisionShape2D children to one body

# One-shot collision toggle
func disable_hitbox_briefly() -> void:
    hitbox_shape.set_deferred("disabled", true)
    await get_tree().create_timer(0.5).timeout
    hitbox_shape.set_deferred("disabled", false)
```

---

## Raycasting

### RayCast2D Node

```gdscript
# Add RayCast2D as child of your character
@onready var ray: RayCast2D = $RayCast2D

func _physics_process(delta: float) -> void:
    if ray.is_colliding():
        var collider := ray.get_collider()       # Node we hit
        var point := ray.get_collision_point()    # Where we hit
        var normal := ray.get_collision_normal()  # Surface normal
        
        if collider is Enemy:
            print("Enemy in line of sight!")
```

### Direct Space Raycasting (No Node Needed)

```gdscript
# Single-frame raycast query
func raycast_from_to(from: Vector2, to: Vector2) -> Dictionary:
    var space_state := get_world_2d().direct_space_state
    var query := PhysicsRayQueryParameters2D.create(from, to)
    
    # Optional: filter
    query.collision_mask = 0b00000011  # Only layers 1 and 2
    query.exclude = [self]             # Don't hit ourselves
    query.collide_with_areas = true    # Also detect Area2D
    
    var result := space_state.intersect_ray(query)
    
    if result:
        # result has: collider, collider_id, normal, position, rid, shape
        print("Hit: ", result.collider.name)
        print("At: ", result.position)
    
    return result

# Line of sight check
func has_line_of_sight(target: Node2D) -> bool:
    var result := raycast_from_to(global_position, target.global_position)
    return result.is_empty() or result.collider == target
```

### 3D Raycasting from Camera (Mouse Picking)

```gdscript
func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseButton and event.pressed:
        var camera := get_viewport().get_camera_3d()
        var from := camera.project_ray_origin(event.position)
        var to := from + camera.project_ray_normal(event.position) * 1000.0
        
        var space := get_world_3d().direct_space_state
        var query := PhysicsRayQueryParameters3D.create(from, to)
        var result := space.intersect_ray(query)
        
        if result:
            print("Clicked on: ", result.collider.name)
            print("At world position: ", result.position)
```

---

## ShapeCast & Physics Queries

### ShapeCast2D — "Fat Raycast"

Like a raycast but sweeps an entire shape:

```gdscript
@onready var shape_cast: ShapeCast2D = $ShapeCast2D

func _physics_process(delta: float) -> void:
    if shape_cast.is_colliding():
        for i in shape_cast.get_collision_count():
            var collider := shape_cast.get_collider(i)
            print("Shape hit: ", collider.name)
```

### Point Query (What's at This Position?)

```gdscript
func get_bodies_at_point(point: Vector2) -> Array:
    var space := get_world_2d().direct_space_state
    var query := PhysicsPointQueryParameters2D.new()
    query.position = point
    query.collision_mask = 0xFFFFFFFF  # All layers
    query.collide_with_bodies = true
    query.collide_with_areas = true
    
    return space.intersect_point(query)  # Array of dictionaries
```

### Shape Query (What Overlaps This Shape?)

```gdscript
func get_bodies_in_circle(center: Vector2, radius: float) -> Array:
    var space := get_world_2d().direct_space_state
    var query := PhysicsShapeQueryParameters2D.new()
    
    var circle := CircleShape2D.new()
    circle.radius = radius
    query.shape = circle
    query.transform = Transform2D(0, center)
    query.collision_mask = 0b00000010  # Layer 2 only
    
    return space.intersect_shape(query)
```

---

## One-Way Platforms

```gdscript
# Method 1: Built-in one_way_collision
# On your StaticBody2D or CollisionShape2D, enable "One Way Collision"
# Player passes through from below but stands on top

# Method 2: Drop through platforms
func _physics_process(delta: float) -> void:
    if Input.is_action_just_pressed("move_down") and is_on_floor():
        # Temporarily disable collision with platforms
        set_collision_mask_value(3, false)  # Disable platform layer
        await get_tree().create_timer(0.3).timeout
        set_collision_mask_value(3, true)   # Re-enable
```

---

## Moving Platforms

```gdscript
# moving_platform.gd
extends AnimatableBody2D

@export var speed: float = 100.0
@export var move_distance: float = 200.0
@export var move_direction: Vector2 = Vector2.RIGHT

var start_position: Vector2
var target_position: Vector2
var going_forward: bool = true

func _ready() -> void:
    start_position = global_position
    target_position = start_position + move_direction.normalized() * move_distance

func _physics_process(delta: float) -> void:
    var target := target_position if going_forward else start_position
    var direction := global_position.direction_to(target)
    
    if global_position.distance_to(target) < 5.0:
        going_forward = not going_forward
    
    # Use velocity for sync_to_physics to work
    velocity = direction * speed
    move_and_slide()
```

> **⚠️ Use AnimatableBody2D, not StaticBody2D!** AnimatableBody correctly conveys velocity to characters standing on it.

---

## Alternative Physics Engines

> **Community consensus:** For 3D projects, switch to Jolt. For 2D-heavy projects with many physics bodies, consider Rapier.

### Jolt Physics (3D)

```
WHAT: AAA-grade 3D physics engine (used in Horizon Forbidden West)
WHY:  130%+ better performance, more stable, multi-core friendly
HOW:  
  • Godot 4.4+: Built-in! No plugin needed.
    Project Settings → Physics → 3D → Physics Engine → "Jolt Physics"
  • Godot 4.0–4.3: Install "Godot Jolt" extension from Asset Library

Benefits:
✅ Dramatically faster with many objects
✅ More accurate collision detection
✅ Objects don't pass through each other as easily
✅ Better stacking behavior

Limitations:
⚠️ 3D only — does not affect 2D physics
⚠️ Some joint properties behave differently
⚠️ WorldBoundaryShape3D has smaller effective size
⚠️ Separate thread mode is experimental
```

### Rapier 2D

```
WHAT: High-performance 2D physics engine written in Rust
WHY:  Significantly faster for scenes with hundreds of 2D bodies
HOW:  Install from Asset Library: "Godot Rapier 2D"
      Project Settings → Physics → 2D → Physics Engine → "Rapier2D"

Best for:
✅ Games with hundreds of simultaneous 2D physics bodies
✅ Complex 2D collision scenarios
✅ When default 2D physics is your profiled bottleneck
```

---

## Physics Performance Tips

```gdscript
# 1. REDUCE PHYSICS TICK RATE
# Project Settings → Physics → Common → Physics Ticks Per Second
# 60 (default) → 30 is fine for most games
# Combine with Physics Interpolation (4.3+) for smooth results

# 2. DISABLE PHYSICS FOR OFF-SCREEN OBJECTS
func _on_screen_exited() -> void:
    set_physics_process(false)
    $CollisionShape2D.set_deferred("disabled", true)

# 3. USE COLLISION LAYERS TO REDUCE PAIR CHECKS
# Each pair of overlapping layers = more checks
# Only enable the exact masks you need

# 4. SIMPLER SHAPES = FASTER
# Circle >> Rectangle >> Capsule >> ConvexPolygon >>> ConcavePolygon
# NEVER use ConcavePolygon on moving objects

# 5. FEWER SOLVER ITERATIONS
# Project Settings → Physics → 2D/3D → Solver Iterations
# Default 16 → try 8 for performance (at cost of accuracy)

# 6. RUN PHYSICS ON SEPARATE THREAD
# Project Settings → Physics → Common → Run on Separate Thread → true
# Frees up main thread for rendering/logic
# ⚠️ Experimental — test thoroughly

# 7. FOR LARGE CROWDS: Skip move_and_slide
# move_and_slide is expensive per-body
# For crowds of 100+, use direct position + grid-based avoidance
```

---

## Physics Gotchas

### The Big List of Physics Mistakes

```gdscript
# ❌ GOTCHA 1: Modifying collision shapes during physics callbacks
# This WILL crash or cause undefined behavior:
func _on_body_entered(body: Node2D) -> void:
    $CollisionShape2D.disabled = true  # CRASH!
# ✅ FIX: Always defer collision changes
func _on_body_entered(body: Node2D) -> void:
    $CollisionShape2D.set_deferred("disabled", true)

# ❌ GOTCHA 2: Using _process() for physics
func _process(delta: float) -> void:
    velocity.x = 100
    move_and_slide()  # Inconsistent at different framerates!
# ✅ FIX: Always use _physics_process()
func _physics_process(delta: float) -> void:
    velocity.x = 100
    move_and_slide()

# ❌ GOTCHA 3: Setting RigidBody position directly
rigid_body.position = Vector2(100, 200)  # Physics engine fights this!
# ✅ FIX: Use forces, impulses, or PhysicsServer

# ❌ GOTCHA 4: Forgetting that collision detection is MUTUAL
# If Player (layer 1, mask 2) detects Enemy (layer 2, mask 0),
# the collision still happens because Player's mask includes Enemy's layer.
# But! body_entered on the Enemy WON'T fire unless Enemy's mask includes Player's layer.

# ❌ GOTCHA 5: Moving children of RigidBody
# Children of RigidBody should NOT be moved independently
# They move with the RigidBody. If you move them, physics breaks.

# ❌ GOTCHA 6: Forgetting to re-enable collision
# If you disable collision temporarily, make SURE you re-enable it!
# Missing this creates invisible, untouchable entities.

# ❌ GOTCHA 7: Area2D gravity override not working
# Make sure your RigidBody has "Gravity" space override set to "Replace" or "Combine"

# ❌ GOTCHA 8: Bouncing on slopes
# CharacterBody bounces when walking down slopes
# ✅ FIX: Set floor_snap_length to something like 8-16 pixels
floor_snap_length = 16.0
```

### Physics Interpolation (Godot 4.3+)

```gdscript
# Enable in Project Settings:
# Physics → Common → Physics Interpolation → enabled

# This smooths physics movement at any framerate
# Without it: jittery at high FPS, stuttery at low FPS
# With it: butter-smooth always

# Per-node control:
node.physics_interpolation_mode = Node.PHYSICS_INTERPOLATION_MODE_ON   # Force on
node.physics_interpolation_mode = Node.PHYSICS_INTERPOLATION_MODE_OFF  # Force off

# Reset interpolation after teleporting:
func teleport(new_pos: Vector2) -> void:
    global_position = new_pos
    reset_physics_interpolation()
```

---

*← [02 — Scene Architecture](./02-scene-architecture.md) | [04 — Animation System](./04-animation-system.md) →*
