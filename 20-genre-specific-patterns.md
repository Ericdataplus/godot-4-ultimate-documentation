# 20 — Genre-Specific Patterns & Recipes

> Battle-tested patterns for specific game types. Nodes to use, pitfalls to avoid, and the architecture that works best for each genre.

---

## Table of Contents

- [2D Platformer](#2d-platformer)
- [Top-Down RPG / Adventure](#top-down-rpg--adventure)
- [3D First-Person](#3d-first-person)
- [3D Third-Person](#3d-third-person)
- [Bullet Hell / Shmup](#bullet-hell--shmup)
- [Tower Defense / Strategy](#tower-defense--strategy)
- [Roguelike / Procedural](#roguelike--procedural)
- [Puzzle / Match-3](#puzzle--match-3)
- [Card Game](#card-game)
- [Visual Novel / Dialogue-Heavy](#visual-novel--dialogue-heavy)
- [Metroidvania](#metroidvania)
- [Fighting Game](#fighting-game)

---

## 2D Platformer

### Recommended Node Setup

```
Root: CharacterBody2D (Player)
├── CollisionShape2D (capsule)
├── AnimatedSprite2D or Sprite2D + AnimationPlayer
├── Camera2D (with smoothing)
├── CoyoteTimer (Timer, one_shot, 0.1-0.15s)
├── JumpBufferTimer (Timer, one_shot, 0.1s)
├── RayCast2D (wall detection — left)
├── RayCast2D (wall detection — right)
└── HitBox (Area2D + CollisionShape2D)
```

### Must-Have Mechanics

```gdscript
# COYOTE TIME — Allow jumping for a short time after leaving a ledge
# Without this, players will constantly complain the jump "doesn't work"
var _was_on_floor: bool = false
@onready var coyote_timer: Timer = $CoyoteTimer

func _physics_process(delta: float) -> void:
    if _was_on_floor and not is_on_floor() and velocity.y >= 0:
        coyote_timer.start()
    _was_on_floor = is_on_floor()
    
    var can_jump := is_on_floor() or not coyote_timer.is_stopped()
```

```gdscript
# JUMP BUFFERING — Register jump input slightly before landing
@onready var buffer_timer: Timer = $JumpBufferTimer

func _physics_process(delta: float) -> void:
    if Input.is_action_just_pressed("jump"):
        if is_on_floor():
            velocity.y = jump_force
        else:
            buffer_timer.start()
    
    if is_on_floor() and not buffer_timer.is_stopped():
        velocity.y = jump_force
        buffer_timer.stop()
```

```gdscript
# VARIABLE JUMP HEIGHT — Release early = shorter jump
if Input.is_action_just_released("jump") and velocity.y < 0:
    velocity.y *= 0.4  # Cut upward velocity
```

```gdscript
# WALL JUMP — Detect walls with raycasts for precision
@onready var ray_left: RayCast2D = $RayLeft
@onready var ray_right: RayCast2D = $RayRight

@export var wall_jump_push: float = 250.0
@export var wall_jump_up: float = -300.0

func _physics_process(delta: float) -> void:
    var on_wall := ray_left.is_colliding() or ray_right.is_colliding()
    
    if Input.is_action_just_pressed("jump") and on_wall and not is_on_floor():
        var push_dir := 1.0 if ray_left.is_colliding() else -1.0
        velocity.x = push_dir * wall_jump_push
        velocity.y = wall_jump_up
        # Lock horizontal input briefly so the push isn't overridden
```

```gdscript
# DASH — Timer-based with cooldown
@export var dash_speed: float = 600.0
@export var dash_duration: float = 0.15
@export var dash_cooldown: float = 0.5

var _is_dashing: bool = false
var _can_dash: bool = true

func dash(direction: Vector2) -> void:
    if not _can_dash or direction == Vector2.ZERO:
        return
    _is_dashing = true
    _can_dash = false
    velocity = direction.normalized() * dash_speed
    
    await get_tree().create_timer(dash_duration).timeout
    _is_dashing = false
    
    await get_tree().create_timer(dash_cooldown).timeout
    _can_dash = true
```

### Platformer Settings Checklist

```
CharacterBody2D properties to configure:
✅ motion_mode = GROUNDED (not Floating!)
✅ floor_snap_length = 8-16 (prevents bouncing on slopes)
✅ floor_stop_on_slope = true (don't slide when idle)
✅ floor_constant_speed = true (same speed up/down slopes)
✅ floor_max_angle = 45° (deg_to_rad(45))

Project Settings:
✅ Gravity matches between project settings and your script
✅ Physics tick rate: 60 (or 30 + interpolation)

Pixel Art specific:
✅ Texture filter: Nearest
✅ Stretch mode: viewport
✅ Snap 2D transforms to pixel: ON
✅ Camera process priority > player process priority
```

---

## Top-Down RPG / Adventure

### Recommended Node Setup

```
Root: CharacterBody2D (Player)
├── CollisionShape2D (small circle at feet — NOT full body)
├── AnimatedSprite2D (8-direction or 4-direction)
├── Camera2D (smoothed, with limits)
├── InteractionArea (Area2D — detects NPCs/objects in range)
├── SortOrigin (set Y-sort to feet, not center)
└── StateMachine (Node — manages idle/walk/attack/interact states)

World Structure:
Root: Node2D (World) — Y Sort Enabled
├── TileMapLayer (ground)
├── TileMapLayer (walls — with collision)
├── TileMapLayer (decorations — above player when behind)
├── NPCs (Node2D — Y Sort Enabled)
├── Items (Node2D)
└── Player
```

### Key Patterns

```gdscript
# INTERACTION SYSTEM — Detect nearest interactable
extends Area2D
class_name InteractionArea

func get_nearest_interactable() -> Node2D:
    var areas := get_overlapping_areas()
    if areas.is_empty():
        return null
    
    var nearest: Node2D = null
    var nearest_dist := INF
    for area in areas:
        if area.owner.has_method("interact"):
            var dist := global_position.distance_squared_to(area.global_position)
            if dist < nearest_dist:
                nearest = area.owner
                nearest_dist = dist
    return nearest

# In player:
func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("interact"):
        var target := interaction_area.get_nearest_interactable()
        if target:
            target.interact(self)
```

```gdscript
# Y-SORTING TRICK — Position collision at feet, not center
# For proper depth ordering in top-down games:
# 1. Enable Y Sort on the parent container
# 2. Put the player's sprite OFFSET upward from the CharacterBody2D origin
# 3. The CharacterBody2D origin stays at the feet
# This makes Y-sort use the feet position for ordering
```

```gdscript
# INVENTORY with Resources
# item_resource.gd
extends Resource
class_name ItemResource

@export var id: String
@export var display_name: String
@export var icon: Texture2D
@export var max_stack: int = 99
@export var item_type: ItemType
@export_multiline var description: String

enum ItemType { CONSUMABLE, WEAPON, ARMOR, KEY, QUEST }
```

```gdscript
# DIALOGUE — Use Dialogic plugin or build with Resources
# Simple dialogue resource approach:
extends Resource
class_name DialogueLine

@export var speaker: String
@export var text: String
@export var portrait: Texture2D
@export var choices: Array[DialogueChoice] = []

extends Resource
class_name DialogueChoice

@export var text: String
@export var next_dialogue: Resource  # Points to next DialogueLine
```

### RPG Tilemap Tips

```
TILEMAP LAYER STRATEGY:
Layer 0: Ground (grass, dirt, water) — no collision
Layer 1: Walls/obstacles — collision enabled  
Layer 2: Above-player decorations (tree tops, roofs)
         Set Z-index > player OR use Y-sort

AUTO-TILING: Use Terrain Sets for natural-looking transitions
- Great for: grass↔dirt, water↔shore, path↔grass
- Setup: TileSet → Terrain Sets → define corners/sides

ANIMATED TILES: Make water, torches, flowers animate
- Import dock → select tile → Animation tab → add frames
```

---

## 3D First-Person

### Recommended Node Setup

```
Root: CharacterBody3D (Player)
├── CollisionShape3D (capsule, height ~1.8)
├── Head (Node3D — Y rotation happens here)
│   └── Camera3D (X rotation happens here)
├── RayCast3D (interaction / shooting)
├── WeaponPivot (Node3D — weapon models)
│   └── SubViewport (for weapon rendering on separate layer)
└── HUD (CanvasLayer)
```

### Key Patterns

```gdscript
# MOUSE LOOK — Split rotation between Head and Camera
@export var mouse_sensitivity: float = 0.002

func _ready() -> void:
    Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseMotion:
        # Y rotation on the BODY (left/right)
        rotate_y(-event.relative.x * mouse_sensitivity)
        # X rotation on the HEAD/CAMERA (up/down)
        head.rotate_x(-event.relative.y * mouse_sensitivity)
        head.rotation.x = clampf(head.rotation.x, deg_to_rad(-89), deg_to_rad(89))
    
    if event.is_action_pressed("ui_cancel"):
        Input.mouse_mode = Input.MOUSE_MODE_VISIBLE
```

```gdscript
# MOVEMENT RELATIVE TO CAMERA FACING
func _physics_process(delta: float) -> void:
    var input := Input.get_vector("move_left", "move_right", "move_forward", "move_back")
    var direction := (transform.basis * Vector3(input.x, 0, input.y)).normalized()
    
    if direction:
        velocity.x = direction.x * speed
        velocity.z = direction.z * speed
    else:
        velocity.x = move_toward(velocity.x, 0, speed)
        velocity.z = move_toward(velocity.z, 0, speed)
    
    # Gravity
    if not is_on_floor():
        velocity.y -= gravity * delta
    
    move_and_slide()
```

```gdscript
# HEAD BOB — Subtle head movement while walking
var _bob_time: float = 0.0
@export var bob_frequency: float = 2.0
@export var bob_amplitude: float = 0.08

func _physics_process(delta: float) -> void:
    if is_on_floor() and velocity.length() > 1.0:
        _bob_time += delta * velocity.length() * 0.1
        camera.position.y = sin(_bob_time * bob_frequency) * bob_amplitude
    else:
        camera.position.y = lerp(camera.position.y, 0.0, delta * 10.0)
```

```gdscript
# FOV SPRINT EFFECT
@export var normal_fov: float = 70.0
@export var sprint_fov: float = 85.0

func _process(delta: float) -> void:
    var target_fov := sprint_fov if is_sprinting else normal_fov
    camera.fov = lerp(camera.fov, target_fov, delta * 8.0)
```

### FPS-Specific Tips

```
⚠️ In Godot 3D, NEGATIVE Z is forward
⚠️ Don't parent Camera3D directly to CharacterBody3D
   → Use a Head (Node3D) pivot between them
⚠️ Use _unhandled_input for mouse look (not _input)
   → Prevents UI elements from stealing mouse events
⚠️ Weapon animations: use SubViewport to render weapon
   on a separate layer so it doesn't clip into walls
⚠️ For stairs: enable floor_snap_length on CharacterBody3D
```

---

## 3D Third-Person

### Recommended Node Setup

```
Root: CharacterBody3D (Player)
├── CollisionShape3D (capsule)
├── PlayerModel (Node3D — the 3D character mesh)
│   ├── AnimationPlayer
│   └── AnimationTree (BlendSpace2D for movement)
├── CameraPivot (Node3D — orbits around player)
│   └── SpringArm3D (collision avoidance!)
│       └── Camera3D
└── HUD (CanvasLayer)
```

### Key Patterns

```gdscript
# SPRINGARM3D — Prevents camera from clipping into walls
# This is THE solution for third-person camera collision
# Configure:
spring_arm.spring_length = 4.0  # Distance from player
spring_arm.collision_mask = 1   # Collide with environment layer only
spring_arm.margin = 0.5         # Slight offset from walls

# Camera orbiting
func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseMotion:
        camera_pivot.rotate_y(-event.relative.x * mouse_sensitivity)
        spring_arm.rotate_x(-event.relative.y * mouse_sensitivity)
        spring_arm.rotation.x = clampf(
            spring_arm.rotation.x, deg_to_rad(-60), deg_to_rad(40)
        )
```

```gdscript
# MOVEMENT RELATIVE TO CAMERA + MODEL ROTATION
func _physics_process(delta: float) -> void:
    var input := Input.get_vector("left", "right", "forward", "back")
    
    # Direction relative to camera, not player
    var cam_basis := camera_pivot.global_transform.basis
    var direction := (cam_basis * Vector3(input.x, 0, input.y)).normalized()
    direction.y = 0  # Flatten to ground plane
    
    if direction:
        velocity.x = direction.x * speed
        velocity.z = direction.z * speed
        # Rotate model to face movement direction
        player_model.rotation.y = lerp_angle(
            player_model.rotation.y,
            atan2(-direction.x, -direction.z),
            delta * 10.0
        )
```

```gdscript
# SEPARATE MODEL FROM CONTROLLER
# Key insight: The CharacterBody3D (controller) should NOT rotate
# Only the visual PlayerModel rotates to face movement direction
# This prevents the camera pivot from spinning with the character
```

---

## Bullet Hell / Shmup

### Recommended Node Setup

```
Root: Node2D (GameWorld)
├── Player (CharacterBody2D or Area2D)
│   ├── CollisionShape2D (tiny circle — just the hitbox)
│   ├── GrazeSensor (Area2D — larger circle for scoring near-misses)
│   └── GunPoints (Marker2D — where bullets spawn)
├── EnemyManager (Node2D)
├── BulletPool (Node — manages all bullets)
├── Background (ParallaxBackground + ParallaxLayers)
└── HUD (CanvasLayer)
```

### Key Patterns

```gdscript
# OBJECT POOLING — Critical for bullet hell performance
# Pre-instantiate hundreds of bullets, reuse instead of create/free

class_name BulletPool
extends Node

var _pool: Array[Node2D] = []
var _bullet_scene: PackedScene

func _ready() -> void:
    _bullet_scene = preload("res://scenes/bullet.tscn")
    for i in 500:
        var bullet := _bullet_scene.instantiate()
        bullet.visible = false
        bullet.set_physics_process(false)
        add_child(bullet)
        _pool.append(bullet)

func get_bullet() -> Node2D:
    for bullet in _pool:
        if not bullet.visible:
            bullet.visible = true
            bullet.set_physics_process(true)
            return bullet
    # Pool exhausted — expand
    var bullet := _bullet_scene.instantiate()
    add_child(bullet)
    _pool.append(bullet)
    return bullet

func return_bullet(bullet: Node2D) -> void:
    bullet.visible = false
    bullet.set_physics_process(false)
```

```gdscript
# BULLET PATTERN SPAWNER
func spawn_circle_pattern(center: Vector2, count: int, speed: float) -> void:
    var angle_step := TAU / count
    for i in count:
        var bullet := pool.get_bullet()
        bullet.global_position = center
        var angle := angle_step * i
        bullet.direction = Vector2(cos(angle), sin(angle))
        bullet.speed = speed

func spawn_spiral_pattern(center: Vector2, arms: int, bullets_per_arm: int) -> void:
    for arm in arms:
        for i in bullets_per_arm:
            var bullet := pool.get_bullet()
            bullet.global_position = center
            var angle := (TAU / arms * arm) + (i * 0.15)
            bullet.direction = Vector2(cos(angle), sin(angle))
            bullet.speed = 150.0 + i * 10.0
```

```gdscript
# SCREEN BOUNDS — Auto-despawn bullets that leave the viewport
# On each bullet:
func _physics_process(delta: float) -> void:
    position += direction * speed * delta
    if not get_viewport_rect().grow(50).has_point(global_position):
        pool.return_bullet(self)
```

### Bullet Hell Tips

```
PERFORMANCE:
• Use Area2D (not CharacterBody2D) for bullets — much lighter
• Object pool: 500+ bullets pre-instantiated
• VisibleOnScreenNotifier2D to skip off-screen processing
• Use GROUPS, not individual references for enemy tracking

GAME FEEL:
• Player hitbox should be TINY (a few pixels)
• Add a "graze" area around the hitbox for near-miss scoring
• Slow-motion focus mode (reduce player speed, show hitbox)
• Iframes after getting hit (0.5-1.0 seconds)
```

---

## Tower Defense / Strategy

### Recommended Node Setup

```
Root: Node2D (Level)
├── Map (TileMapLayer — placement grid)
├── Paths (Node2D)
│   └── Path2D + PathFollow2D (enemy routes)
├── TowerSlots (Node2D — valid placement positions)
├── Enemies (Node2D — spawned enemies)
├── Projectiles (Node2D)
├── WaveManager (Node — controls spawning)
└── HUD (CanvasLayer — build menu, wave info, resources)
```

### Key Patterns

```gdscript
# TOWER TARGETING — Find nearest/first/strongest enemy in range
extends Area2D
class_name Tower

@export var damage: float = 10.0
@export var fire_rate: float = 1.0
@export var target_mode: TargetMode = TargetMode.NEAREST

enum TargetMode { NEAREST, FIRST, STRONGEST, WEAKEST }

func get_target() -> Node2D:
    var enemies := get_overlapping_bodies().filter(
        func(b): return b.is_in_group("enemy") and b.health > 0
    )
    if enemies.is_empty():
        return null
    
    match target_mode:
        TargetMode.NEAREST:
            return enemies.reduce(func(best, e):
                return e if global_position.distance_squared_to(e.global_position) < global_position.distance_squared_to(best.global_position) else best
            )
        TargetMode.STRONGEST:
            return enemies.reduce(func(best, e):
                return e if e.health > best.health else best
            )
        _:
            return enemies[0]
```

```gdscript
# ENEMY PATHING — Using Path2D + PathFollow2D
extends PathFollow2D

@export var speed: float = 100.0
var health: float = 100.0

func _physics_process(delta: float) -> void:
    progress += speed * delta
    
    # Reached the end
    if progress_ratio >= 1.0:
        reached_base.emit()
        queue_free()
```

```gdscript
# WAVE SYSTEM — Data-driven with Resources
extends Resource
class_name WaveData

@export var enemies: Array[EnemySpawn] = []
@export var time_between_spawns: float = 0.5
@export var reward_gold: int = 50

extends Resource
class_name EnemySpawn

@export var scene: PackedScene
@export var count: int = 1
@export var spawn_delay: float = 0.3
```

```gdscript
# GRID-BASED PLACEMENT
func world_to_grid(world_pos: Vector2) -> Vector2i:
    return Vector2i(world_pos / tile_size)

func grid_to_world(grid_pos: Vector2i) -> Vector2:
    return Vector2(grid_pos) * tile_size + Vector2(tile_size / 2, tile_size / 2)

func can_place_tower(grid_pos: Vector2i) -> bool:
    return not _occupied_cells.has(grid_pos) and _buildable_cells.has(grid_pos)
```

---

## Roguelike / Procedural

### Procedural Generation Approaches

```
ALGORITHM COMPARISON:
┌────────────────────┬──────────────────────┬──────────────┐
│ Algorithm          │ Best For             │ Godot Tool   │
├────────────────────┼──────────────────────┼──────────────┤
│ Random Walker      │ Cave-like, Spelunky  │ TileMapLayer │
│ BSP (Binary Space) │ Rectangle rooms      │ TileMapLayer │
│ Cellular Automata  │ Organic caves        │ TileMapLayer │
│ Room + Corridors   │ Traditional dungeon  │ Scenes       │
│ Wave Function      │ Constrained layouts  │ TileMapLayer │
│ Perlin/Simplex     │ Overworld terrain    │ Image/Shader │
└────────────────────┴──────────────────────┴──────────────┘
```

```gdscript
# RANDOM WALKER — Simple but effective for caves
func generate_cave(width: int, height: int, steps: int) -> Array:
    var map: Array = []
    for x in width:
        map.append([])
        for y in height:
            map[x].append(1)  # 1 = wall
    
    var pos := Vector2i(width / 2, height / 2)
    var directions := [Vector2i.UP, Vector2i.DOWN, Vector2i.LEFT, Vector2i.RIGHT]
    
    for i in steps:
        map[pos.x][pos.y] = 0  # 0 = floor
        pos += directions.pick_random()
        pos.x = clampi(pos.x, 1, width - 2)
        pos.y = clampi(pos.y, 1, height - 2)
    
    return map
```

```gdscript
# ROOM-BASED DUNGEON GENERATION
func generate_rooms(room_count: int, min_size: Vector2i, max_size: Vector2i) -> Array[Rect2i]:
    var rooms: Array[Rect2i] = []
    var attempts := 0
    
    while rooms.size() < room_count and attempts < 1000:
        var size := Vector2i(
            randi_range(min_size.x, max_size.x),
            randi_range(min_size.y, max_size.y)
        )
        var pos := Vector2i(
            randi_range(0, map_width - size.x),
            randi_range(0, map_height - size.y)
        )
        var room := Rect2i(pos, size)
        
        # Check overlap
        var overlap := false
        for existing in rooms:
            if room.grow(2).intersects(existing):
                overlap = true
                break
        
        if not overlap:
            rooms.append(room)
        attempts += 1
    
    return rooms

# Connect rooms with L-shaped corridors
func connect_rooms(room_a: Rect2i, room_b: Rect2i) -> void:
    var center_a := room_a.get_center()
    var center_b := room_b.get_center()
    
    # Horizontal then vertical
    for x in range(mini(center_a.x, center_b.x), maxi(center_a.x, center_b.x) + 1):
        carve_tile(Vector2i(x, center_a.y))
    for y in range(mini(center_a.y, center_b.y), maxi(center_a.y, center_b.y) + 1):
        carve_tile(Vector2i(center_b.x, y))
```

### Roguelike Architecture Tips

```
ITEM/ABILITY SYSTEM:
✅ Use Resources for all items, abilities, status effects
✅ Define base stats + modifiers (stacking, multiplicative)
✅ Use signals for on_pickup, on_use, on_proc events

SEED SYSTEM:
✅ Store and display the RNG seed for shareable runs
✅ Use seed for ALL generation (not just map layout)

ROOM INSTANCING:
✅ Make each room a separate scene
✅ Randomize room contents separately from layout
✅ Pre-fabricated "special" rooms (shops, boss, treasure)
✅ Gaea addon for streamlined procedural generation
```

---

## Puzzle / Match-3

### Key Patterns

```gdscript
# GRID DATA STRUCTURE — The core of any puzzle game
class_name PuzzleGrid
extends Node2D

@export var grid_size: Vector2i = Vector2i(8, 8)
@export var cell_size: float = 64.0

var _grid: Array = []  # 2D array of piece types (int or null)

func _ready() -> void:
    _grid.resize(grid_size.x)
    for x in grid_size.x:
        _grid[x] = []
        _grid[x].resize(grid_size.y)
        for y in grid_size.y:
            _grid[x][y] = randi_range(0, piece_types - 1)

func grid_to_world(grid_pos: Vector2i) -> Vector2:
    return Vector2(grid_pos) * cell_size + Vector2(cell_size / 2, cell_size / 2)

func world_to_grid(world_pos: Vector2) -> Vector2i:
    return Vector2i(world_pos / cell_size)
```

```gdscript
# MATCH DETECTION — Check rows and columns
func find_matches() -> Array[Array]:
    var matches: Array[Array] = []
    
    # Check horizontal
    for y in grid_size.y:
        var run: Array[Vector2i] = [Vector2i(0, y)]
        for x in range(1, grid_size.x):
            if _grid[x][y] == _grid[x - 1][y] and _grid[x][y] != -1:
                run.append(Vector2i(x, y))
            else:
                if run.size() >= 3:
                    matches.append(run.duplicate())
                run = [Vector2i(x, y)]
        if run.size() >= 3:
            matches.append(run)
    
    # Check vertical (same logic, swap x/y)
    # ...
    
    return matches
```

```gdscript
# GRAVITY — Make pieces fall into empty spaces
func apply_gravity() -> void:
    for x in grid_size.x:
        var write_y := grid_size.y - 1
        for y in range(grid_size.y - 1, -1, -1):
            if _grid[x][y] != -1:
                _grid[x][write_y] = _grid[x][y]
                if write_y != y:
                    _grid[x][y] = -1
                    # Animate the piece falling
                write_y -= 1
        # Fill empty spaces at top with new pieces
        for y in range(write_y, -1, -1):
            _grid[x][y] = randi_range(0, piece_types - 1)
```

### Puzzle Tips

```
VISUAL FEEL:
• Use Node2D for pieces (not Control) — easier to animate
• Tween pieces falling / swapping for juice
• Add particle effects on match
• Screen shake on large combos

ARCHITECTURE:
• Separate LOGIC (grid data) from VISUALS (sprite nodes)
• Use inheritance for piece types (base piece → fire/water/etc.)
• State machine: IDLE → SWAPPING → MATCHING → FALLING → IDLE
```

---

## Card Game

### Key Patterns

```gdscript
# CARD DATA as Resource
extends Resource
class_name CardData

@export var id: String
@export var card_name: String
@export var description: String
@export var cost: int
@export var card_type: CardType
@export var art: Texture2D
@export var effects: Array[CardEffect] = []

enum CardType { ATTACK, DEFENSE, SKILL, ITEM }
```

```gdscript
# HAND LAYOUT — Fan cards with smooth repositioning
func arrange_hand() -> void:
    var card_count := hand.size()
    var total_width := minf(card_count * card_spacing, max_hand_width)
    var start_x := -total_width / 2.0
    
    for i in card_count:
        var card: Node2D = hand[i]
        var target_x := start_x + (total_width / maxi(card_count - 1, 1)) * i
        var target_y := abs(target_x) * 0.05  # Arc shape
        var target_rot := -target_x * 0.001   # Slight tilt
        
        var tween := card.create_tween().set_parallel(true)
        tween.tween_property(card, "position:x", target_x, 0.2)
        tween.tween_property(card, "position:y", target_y, 0.2)
        tween.tween_property(card, "rotation", target_rot, 0.2)
```

```gdscript
# DRAG AND DROP for cards
func _on_card_gui_input(event: InputEvent, card: Node2D) -> void:
    if event is InputEventMouseButton:
        if event.pressed:
            _dragging_card = card
            _drag_offset = card.global_position - event.global_position
        else:
            _drop_card(_dragging_card)
            _dragging_card = null
    
    if event is InputEventMouseMotion and _dragging_card:
        _dragging_card.global_position = event.global_position + _drag_offset
```

### Card Game Tips

```
ARCHITECTURE:
✅ Separate card DATA (Resource) from card VISUALS (Scene)
✅ Use signals for: card_played, card_drawn, card_discarded
✅ Deck = Array[CardData], shuffle with .shuffle()
✅ Discard pile = another Array, reshuffle when deck is empty
✅ For multiplayer: only the server shuffles/deals

JUICE:
✅ Cards hover/scale up when mouse is over them
✅ Smooth fan layout with tweens
✅ Draw animation: cards fly from deck to hand
✅ Play animation: card flies to play area, then resolves
```

---

## Visual Novel / Dialogue-Heavy

### Recommended Setup

```
Root: Control (FullRect)
├── BackgroundLayer (TextureRect — scene backgrounds)
├── CharacterLayer (Node2D)
│   ├── LeftCharacter (Sprite2D)
│   ├── CenterCharacter (Sprite2D)
│   └── RightCharacter (Sprite2D)
├── DialogueBox (PanelContainer)
│   ├── SpeakerLabel (Label)
│   ├── TextDisplay (RichTextLabel — for typewriter effect)
│   └── ChoiceContainer (VBoxContainer — branching choices)
└── UILayer (CanvasLayer)
    ├── SaveLoadMenu
    ├── SettingsMenu
    └── BacklogButton
```

### Key Patterns

```gdscript
# TYPEWRITER EFFECT — Reveal text character by character
func display_text(text: String, speed: float = 0.03) -> void:
    text_label.text = text
    text_label.visible_ratio = 0.0
    
    var tween := create_tween()
    tween.tween_property(text_label, "visible_ratio", 1.0, text.length() * speed)
    
    # Click to skip
    _text_complete = false
    tween.finished.connect(func(): _text_complete = true)

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("advance"):
        if not _text_complete:
            text_label.visible_ratio = 1.0
            _text_complete = true
        else:
            advance_dialogue()
```

```gdscript
# CHARACTER TRANSITIONS
func show_character(slot: String, texture: Texture2D, emotion: String = "neutral") -> void:
    var sprite: Sprite2D = get_node("CharacterLayer/" + slot)
    sprite.texture = texture
    
    var tween := create_tween()
    tween.tween_property(sprite, "modulate:a", 1.0, 0.3)

func dim_inactive_speakers(active_slot: String) -> void:
    for child in $CharacterLayer.get_children():
        var target_color := Color.WHITE if child.name == active_slot else Color(0.6, 0.6, 0.6)
        var tween := child.create_tween()
        tween.tween_property(child, "modulate", target_color, 0.2)
```

### Visual Novel Tips

```
PLUGINS:
✅ Dialogic 2 — Full-featured VN/dialogue plugin for Godot 4
   Handles: timelines, characters, choices, save/load, styling
✅ For simpler needs, build your own with Resources

MUST-HAVES:
✅ Text skip (hold to fast-forward)
✅ Auto-advance mode (with adjustable delay)
✅ Dialogue backlog (scroll through past text)
✅ Save anywhere (serialize current dialogue position)
✅ Multiple save slots
```

---

## Metroidvania

### Key Patterns

```gdscript
# ABILITY GATING — Track which abilities the player has
# autoload: player_abilities.gd
extends Node

var abilities: Dictionary = {
    "double_jump": false,
    "wall_jump": false,
    "dash": false,
    "wall_climb": false,
    "swim": false,
    "grapple": false,
}

signal ability_unlocked(ability_name: String)

func unlock(ability: String) -> void:
    abilities[ability] = true
    ability_unlocked.emit(ability)

func has_ability(ability: String) -> bool:
    return abilities.get(ability, false)
```

```gdscript
# MAP REVEAL SYSTEM — Fog of war on minimap
# Each room is a Rect2 defining its bounds
# As player enters a room, mark it as "visited"

var _visited_rooms: Array[String] = []
var _room_data: Dictionary = {}  # room_id → Rect2

func enter_room(room_id: String) -> void:
    if room_id not in _visited_rooms:
        _visited_rooms.append(room_id)
        map_updated.emit()
```

```gdscript
# SAVE POINTS scattered throughout the world
# Save: player position, abilities, visited rooms, defeated bosses
func save_state() -> Dictionary:
    return {
        "position": {"x": global_position.x, "y": global_position.y},
        "abilities": PlayerAbilities.abilities.duplicate(),
        "visited_rooms": _visited_rooms.duplicate(),
        "defeated_bosses": _defeated_bosses.duplicate(),
        "health": health,
        "max_health": max_health,
    }
```

### Metroidvania Tips

```
MAP DESIGN:
✅ Design the map on paper FIRST before building
✅ Each ability should open 2-3 new paths minimum
✅ Place ability upgrades in dead-end rooms (reward exploration)
✅ Use color-coded doors/barriers for gating clarity
✅ Breadcrumb items lead players toward the intended path

TECHNICAL:
✅ Use TileMapLayer for individual rooms, instanced sub-scenes
✅ Room transitions: fade to black, load adjacent room
✅ Camera limits per room (set Camera2D.limit_*)
✅ Persistent room state (enemies stay dead until reset)
```

---

## Fighting Game

### Key Patterns

```gdscript
# INPUT BUFFER — Critical for combo execution
class_name InputBuffer
extends Node

var _buffer: Array[Dictionary] = []
@export var buffer_window: float = 0.15  # seconds

func record_input(action: String) -> void:
    _buffer.append({
        "action": action,
        "time": Time.get_ticks_msec() / 1000.0,
    })

func check_sequence(sequence: Array[String]) -> bool:
    var now := Time.get_ticks_msec() / 1000.0
    # Clean old inputs
    _buffer = _buffer.filter(func(i): return now - i.time < 1.0)
    
    # Check if sequence exists in buffer in order
    var seq_idx := 0
    for input in _buffer:
        if input.action == sequence[seq_idx]:
            seq_idx += 1
            if seq_idx >= sequence.size():
                return true
    return false
```

```gdscript
# HITBOX/HURTBOX SYSTEM
# Separate hitbox (attack) from hurtbox (vulnerable area)
# They should be on separate collision layers!

# On attack:
func enable_attack_hitbox(frames: int = 3) -> void:
    $HitBox/CollisionShape2D.set_deferred("disabled", false)
    await get_tree().create_timer(frames / 60.0).timeout
    $HitBox/CollisionShape2D.set_deferred("disabled", true)
```

### Fighting Game Tips

```
INPUT:
✅ Input buffer for command inputs (quarter circles, DPs)
✅ Store direction relative to facing (forward/back, not left/right)
✅ Rollback netcode for online play (complex but necessary)

FEEL:
✅ Hitstop on EVERY hit (freeze both players for 2-4 frames)
✅ Push-back on block
✅ Different hit effects for light/medium/heavy attacks
✅ Camera zoom on critical hits

ARCHITECTURE:
✅ State machine with strict transitions
✅ Frame-based timing (not delta-based)
✅ Separate hitbox/hurtbox collision layers
```

---

## Cross-Genre Quick Reference

```
WHICH PHYSICS BODY TO USE:
├── Player character → CharacterBody2D/3D
├── Enemies (simple) → CharacterBody2D/3D
├── Enemies (swarm) → Area2D + direct position
├── Projectiles → Area2D (lightweight)
├── Physics objects → RigidBody2D/3D
├── Walls/terrain → StaticBody2D/3D
├── Moving platforms → AnimatableBody2D/3D
├── Pickups/triggers → Area2D/3D
└── Puzzle pieces → Area2D or Control

WHICH CAMERA TO USE:
├── 2D Platformer → Camera2D (smoothed, with limits)
├── 2D Top-Down → Camera2D (smoothed, Y-sort)
├── 3D First-Person → Camera3D (child of Head pivot)
├── 3D Third-Person → Camera3D (in SpringArm3D)
├── Side-scroller → Camera2D (drag margins)
└── Fixed camera → Camera2D/3D (static, no follow)
```

---

*← [19 — Modular Architecture](./19-modular-architecture.md) | [README](./README.md) →*
