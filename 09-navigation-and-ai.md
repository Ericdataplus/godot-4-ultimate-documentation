# 09 — Navigation & AI

> Pathfinding, obstacle avoidance, and AI behavior patterns.

---

## Table of Contents

- [Navigation Setup](#navigation-setup)
- [NavigationAgent2D](#navigationagent2d)
- [NavigationAgent3D](#navigationagent3d)
- [Obstacle Avoidance](#obstacle-avoidance)
- [Dynamic Navigation](#dynamic-navigation)
- [AI Behavior Patterns](#ai-behavior-patterns)
- [Behavior Trees](#behavior-trees)

---

## Navigation Setup

### 2D Navigation

```
1. Add NavigationRegion2D to your scene
2. Add a NavigationPolygon resource
3. Draw the walkable area polygon
4. OR use "Bake NavigationMesh" for auto-generation from collision shapes
```

```gdscript
# Auto-bake on ready (useful for procedural levels)
@onready var nav_region: NavigationRegion2D = $NavigationRegion2D

func _ready() -> void:
    nav_region.bake_navigation_polygon()
```

### 3D Navigation

```
1. Add NavigationRegion3D to your scene
2. Add a NavigationMesh resource
3. Set agent properties (height, radius, max slope)
4. Click "Bake NavigationMesh" in toolbar
```

---

## NavigationAgent2D

### Basic Pathfinding Enemy

```gdscript
extends CharacterBody2D

@export var speed: float = 100.0
@onready var nav_agent: NavigationAgent2D = $NavigationAgent2D

var target: Node2D

func _ready() -> void:
    # Wait for navigation map to sync
    await get_tree().physics_frame
    
    nav_agent.path_desired_distance = 4.0      # How close to path points
    nav_agent.target_desired_distance = 8.0     # How close to final target
    nav_agent.max_speed = speed

func set_target(new_target: Node2D) -> void:
    target = new_target

func _physics_process(delta: float) -> void:
    if target == null:
        return
    
    nav_agent.target_position = target.global_position
    
    if nav_agent.is_navigation_finished():
        return
    
    var next_pos := nav_agent.get_next_path_position()
    var direction := global_position.direction_to(next_pos)
    
    velocity = direction * speed
    move_and_slide()
```

### Advanced Navigation Features

```gdscript
# Check if target is reachable
if nav_agent.is_target_reachable():
    # Path exists
    pass

# Get remaining distance
var remaining := nav_agent.distance_to_target()

# Path changed signal
nav_agent.path_changed.connect(func():
    print("Path recalculated!")
)

# Target reached signal
nav_agent.target_reached.connect(func():
    print("Arrived at destination!")
)

# Navigation link signal (teleporters, ladders)
nav_agent.link_reached.connect(func(details: Dictionary):
    # Handle special navigation (e.g., climb ladder)
    pass
)

# Debug visualization
nav_agent.debug_enabled = true
nav_agent.debug_path_custom_color = Color.RED
```

---

## NavigationAgent3D

```gdscript
extends CharacterBody3D

@export var speed: float = 5.0
@onready var nav_agent: NavigationAgent3D = $NavigationAgent3D

var gravity: float = ProjectSettings.get_setting("physics/3d/default_gravity")

func _ready() -> void:
    await get_tree().physics_frame
    nav_agent.max_speed = speed
    nav_agent.path_desired_distance = 0.5
    nav_agent.target_desired_distance = 1.0

func set_target_position(pos: Vector3) -> void:
    nav_agent.target_position = pos

func _physics_process(delta: float) -> void:
    if nav_agent.is_navigation_finished():
        velocity = Vector3.ZERO
        move_and_slide()
        return
    
    var next_pos := nav_agent.get_next_path_position()
    var direction := global_position.direction_to(next_pos)
    direction.y = 0  # Keep movement horizontal
    direction = direction.normalized()
    
    velocity.x = direction.x * speed
    velocity.z = direction.z * speed
    
    # Apply gravity
    if not is_on_floor():
        velocity.y -= gravity * delta
    
    move_and_slide()
    
    # Face movement direction
    if direction.length() > 0.1:
        look_at(global_position + direction, Vector3.UP)
```

---

## Obstacle Avoidance

Avoidance uses RVO (Reciprocal Velocity Obstacle) to prevent agent collisions.

```gdscript
# Enable avoidance on NavigationAgent2D
func _ready() -> void:
    nav_agent.avoidance_enabled = true
    nav_agent.radius = 16.0              # Agent collision radius
    nav_agent.max_speed = speed
    nav_agent.neighbor_distance = 200.0  # Detection range
    nav_agent.max_neighbors = 10
    
    # Connect the safe velocity signal
    nav_agent.velocity_computed.connect(_on_velocity_computed)

func _physics_process(delta: float) -> void:
    if nav_agent.is_navigation_finished():
        nav_agent.velocity = Vector2.ZERO
        return
    
    var next_pos := nav_agent.get_next_path_position()
    var direction := global_position.direction_to(next_pos)
    var desired_velocity := direction * speed
    
    # Send desired velocity — avoidance calculates safe velocity
    nav_agent.velocity = desired_velocity

func _on_velocity_computed(safe_velocity: Vector2) -> void:
    velocity = safe_velocity
    move_and_slide()
```

### NavigationObstacle (Dynamic Obstacles)

```gdscript
# Add NavigationObstacle2D/3D for moving objects agents should avoid
# (e.g., a moving barrel, a swinging door)

# The obstacle pushes agents away without recalculating the nav mesh
var obstacle: NavigationObstacle2D
obstacle.radius = 32.0
obstacle.avoidance_enabled = true
obstacle.velocity = Vector2(50, 0)  # Moving obstacle
```

---

## Dynamic Navigation

### Runtime NavMesh Modification

```gdscript
# Rebuild navigation when level changes
func on_wall_destroyed(wall: Node2D) -> void:
    wall.queue_free()
    # Re-bake after removing the wall
    await get_tree().process_frame
    nav_region.bake_navigation_polygon()

# Multiple navigation regions that combine
# Simply add multiple NavigationRegion2D nodes — they merge automatically

# Navigation layers (different agent types)
# Layer 1: Ground units
# Layer 2: Flying units
# Layer 3: Aquatic units
nav_agent.navigation_layers = 1  # Only walk on ground
```

---

## AI Behavior Patterns

### Patrol Pattern

```gdscript
extends CharacterBody2D

@export var patrol_points: Array[Marker2D] = []
@onready var nav_agent: NavigationAgent2D = $NavigationAgent2D

var current_patrol_index: int = 0
var speed: float = 60.0

func _ready() -> void:
    await get_tree().physics_frame
    if not patrol_points.is_empty():
        nav_agent.target_position = patrol_points[0].global_position

func _physics_process(delta: float) -> void:
    if nav_agent.is_navigation_finished():
        _advance_patrol()
        return
    
    var next_pos := nav_agent.get_next_path_position()
    velocity = global_position.direction_to(next_pos) * speed
    move_and_slide()

func _advance_patrol() -> void:
    current_patrol_index = (current_patrol_index + 1) % patrol_points.size()
    nav_agent.target_position = patrol_points[current_patrol_index].global_position
```

### Chase & Lose Pattern

```gdscript
enum AIState { PATROL, CHASE, SEARCH, RETURN }

var state: AIState = AIState.PATROL
var player: Node2D
var detection_range: float = 200.0
var lose_range: float = 400.0
var last_known_position: Vector2
var search_timer: float = 0.0

func _physics_process(delta: float) -> void:
    _detect_player()
    
    match state:
        AIState.PATROL:
            _patrol(delta)
        AIState.CHASE:
            _chase(delta)
        AIState.SEARCH:
            _search(delta)
        AIState.RETURN:
            _return_to_patrol(delta)

func _detect_player() -> void:
    if player == null:
        return
    var dist := global_position.distance_to(player.global_position)
    
    match state:
        AIState.PATROL:
            if dist < detection_range and _has_line_of_sight():
                state = AIState.CHASE
        AIState.CHASE:
            if dist > lose_range or not _has_line_of_sight():
                last_known_position = player.global_position
                state = AIState.SEARCH
                search_timer = 5.0

func _chase(delta: float) -> void:
    nav_agent.target_position = player.global_position
    var next := nav_agent.get_next_path_position()
    velocity = global_position.direction_to(next) * speed * 1.5
    move_and_slide()

func _search(delta: float) -> void:
    nav_agent.target_position = last_known_position
    search_timer -= delta
    
    if search_timer <= 0:
        state = AIState.RETURN
    
    if nav_agent.is_navigation_finished():
        # Look around...
        pass
    else:
        var next := nav_agent.get_next_path_position()
        velocity = global_position.direction_to(next) * speed * 0.8
        move_and_slide()

func _has_line_of_sight() -> bool:
    var space := get_world_2d().direct_space_state
    var query := PhysicsRayQueryParameters2D.create(
        global_position, player.global_position
    )
    query.exclude = [self]
    query.collision_mask = 0b0100  # Environment only
    var result := space.intersect_ray(query)
    return result.is_empty()
```

---

## Behavior Trees

A more scalable alternative to state machines for complex AI:

```gdscript
# bt_node.gd — Base behavior tree node
extends Node
class_name BTNode

enum Status { SUCCESS, FAILURE, RUNNING }

func tick(actor: Node, blackboard: Dictionary) -> Status:
    return Status.FAILURE

# bt_selector.gd — Tries children until one succeeds
class_name BTSelector extends BTNode

func tick(actor: Node, blackboard: Dictionary) -> Status:
    for child in get_children():
        if child is BTNode:
            var result := child.tick(actor, blackboard)
            if result != Status.FAILURE:
                return result
    return Status.FAILURE

# bt_sequence.gd — Runs children in order, fails if any fail
class_name BTSequence extends BTNode

func tick(actor: Node, blackboard: Dictionary) -> Status:
    for child in get_children():
        if child is BTNode:
            var result := child.tick(actor, blackboard)
            if result != Status.SUCCESS:
                return result
    return Status.SUCCESS
```

### Example: Enemy BT

```
BTSelector (root)
├── BTSequence "Attack"
│   ├── IsPlayerInRange (condition)
│   └── AttackPlayer (action)
├── BTSequence "Chase"
│   ├── IsPlayerDetected (condition)
│   └── ChasePlayer (action)
└── Patrol (action/default)
```

```gdscript
# is_player_in_range.gd
extends BTNode

@export var range: float = 50.0

func tick(actor: Node, blackboard: Dictionary) -> Status:
    var player = blackboard.get("player")
    if player and actor.global_position.distance_to(player.global_position) < range:
        return Status.SUCCESS
    return Status.FAILURE
```

---

*← [08 — 3D Rendering](./08-3d-rendering.md) | [10 — Networking](./10-networking.md) →*
