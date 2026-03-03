# 19 — Modular Architecture for AI-Assisted Development

> How to structure your Godot project so AI agents can build in it effectively. Community-proven patterns, React-like composability, and the hard-won lessons of real Godot developers.

---

## Table of Contents

- [Why This Matters](#why-this-matters)
- [The Core Philosophy](#the-core-philosophy)
- [Call Down, Signal Up — The Golden Rule](#call-down-signal-up)
- [The Root Mediator Pattern](#the-root-mediator-pattern)
- [Resource-as-Data-Bus](#resource-as-data-bus)
- [Composition + Inheritance — Know When to Use Each](#composition-vs-inheritance)
- [Atomic Design for Godot](#atomic-design-for-godot)
- [The Component Contract](#the-component-contract)
- [Self-Contained Scene Pattern](#self-contained-scene-pattern)
- [Vertical Slice Folder Structure](#vertical-slice-folder-structure)
- [The Event Bus (Global Signal Hub)](#the-event-bus)
- [Dependency Injection Patterns](#dependency-injection)
- [Callable Injection — Advanced Behavior Swapping](#callable-injection)
- [Data-Driven Architecture](#data-driven-architecture)
- [The Registry Pattern](#the-registry-pattern)
- [Unit Testing with GUT](#unit-testing-with-gut)
- [Making Cross-Project Plugins](#cross-project-plugins)
- [File Conventions for AI](#file-conventions-for-ai)
- [Feature Module Boundaries](#feature-module-boundaries)
- [Practical Template](#practical-template)
- [Anti-Patterns to Avoid](#anti-patterns)
- [Modularity Checklist](#modularity-checklist)

---

## Why This Matters

When an AI agent (or any developer) needs to add features to your Godot project, **modularity determines success**. A monolithic 2000-line player script is nearly impossible to safely modify. But a project built from small, self-contained, well-documented components? An AI can:

- **Add** new components without breaking existing ones
- **Replace** individual pieces without understanding the whole
- **Test** components in isolation
- **Understand** the codebase from file/folder names alone
- **Compose** new entities by mixing existing components

This is the same insight that made React dominant: **composition > inheritance**, **props > globals**, **one-way data flow > spaghetti**.

---

## The Core Philosophy

### React Principles → Godot Equivalents

| React Concept | Godot Equivalent |
|---------------|-----------------|
| Components | Scenes (.tscn + .gd) |
| Props | `@export` variables |
| State | Private variables in script |
| Events / Callbacks | Signals |
| Context | Autoloads (sparingly) |
| Hooks | Component nodes (child patterns) |
| Children / Slots | Node children + `@export NodePath` |
| Render | `_process()` + `_draw()` |
| One-way data flow | Signals UP, Calls DOWN |
| Composition | Scene instancing |
| Keys / Identity | Node names + groups |

### The Three Rules

```
1. EVERY REUSABLE BEHAVIOR = ONE SCENE + ONE SCRIPT
   (like a React component = one file)

2. INPUTS VIA @EXPORT, OUTPUTS VIA SIGNALS
   (like props in, events out)

3. NO REACHING UP THE TREE — EVER
   (no get_parent(), no get_node("../../"), no assumptions)
```

---

## Call Down, Signal Up

> **This is the single most important pattern in Godot.** The community has universally agreed on it. GDQuest calls it the "golden rule." Every experienced Godot developer follows it.

### The Rule

```
PARENT → CHILD:  Direct method calls are OK
                 (Parents KNOW their children)

CHILD → PARENT:  Emit signals ONLY
                 (Children must NOT know who their parent is)

SIBLING → SIBLING: Go through the parent (mediator)
                    (Siblings should NOT know about each other)
```

### Why This Works

```gdscript
# ✅ CALLING DOWN — Parent calls child's method directly
# This is safe because the parent CHOSE to add this child
func _on_enemy_spotted() -> void:
    $WeaponComponent.attack()       # Parent knows its children
    $AnimationPlayer.play("attack") # Direct call = fine

# ✅ SIGNALING UP — Child emits signal, parent (or anyone) listens
# health_component.gd
signal died

func take_damage(amount: int) -> void:
    _health -= amount
    if _health <= 0:
        died.emit()  # "I don't know who cares, but I'm dead"

# Parent connects:
func _ready() -> void:
    $HealthComponent.died.connect(_on_entity_died)

# ❌ ANTI-PATTERN — Child reaching up for parent
func take_damage(amount: int) -> void:
    _health -= amount
    if _health <= 0:
        get_parent().die()  # NEVER DO THIS
        # What if this component is moved to a different parent?
        # What if no parent has die()? Crash!
```

### The Communication Spectrum

```
MOST COUPLED ←──────────────────────────────→ LEAST COUPLED

get_parent()    get_node()    Direct call    Signal    EventBus    Resource Bus
   ❌ Never       ⚠️ Avoid     ✅ Down only   ✅ Up     ✅ Global   ✅ Shared data
```

---

## The Root Mediator Pattern

> Popularized by GDQuest. When siblings need to communicate, the **root node of the entity acts as the mediator** — it wires children together.

```gdscript
# enemy.gd — The root node is the "grand mediator"
extends CharacterBody2D

@onready var health: HealthComponent = $HealthComponent
@onready var ai: AIComponent = $AIComponent
@onready var sprite: AnimatedSprite2D = $AnimatedSprite2D
@onready var hitbox: HitboxComponent = $HitboxComponent

func _ready() -> void:
    # Root node wires sibling communication
    hitbox.hit_received.connect(_on_hit_received)
    health.died.connect(_on_died)
    health.health_changed.connect(_on_health_changed)

func _on_hit_received(damage: int, source: Node) -> void:
    health.take_damage(damage)         # Call DOWN to health
    sprite.play("hurt")                # Call DOWN to sprite
    ai.alert(source.global_position)   # Call DOWN to AI

func _on_died() -> void:
    sprite.play("death")               # Call DOWN
    ai.disable()                       # Call DOWN
    await sprite.animation_finished
    queue_free()

func _on_health_changed(current: int, max_hp: int) -> void:
    if current < max_hp * 0.25:
        ai.set_behavior("flee")        # Call DOWN — root decides
```

### Why This is Powerful

- **HealthComponent** doesn't know AIComponent exists
- **AIComponent** doesn't know HealthComponent exists
- **HitboxComponent** doesn't know what happens when something is hit
- **Only the root** knows how its children relate — and it's ONE place to look
- **Each component** can be dropped into a totally different entity

---

## Resource-as-Data-Bus

> This community-discovered pattern uses Godot's `Resource` system to create **shared data objects** that multiple nodes can read/write without knowing about each other. It eliminates the "signal cascade" problem where signals have to bubble up through 5 layers of parents.

```gdscript
# shared_health_data.gd — A Resource that acts as a data bus
extends Resource
class_name SharedHealthData

signal health_changed(current: int, maximum: int)
signal died

@export var max_health: int = 100
var _current_health: int

var current_health: int:
    get: return _current_health
    set(value):
        _current_health = clampi(value, 0, max_health)
        health_changed.emit(_current_health, max_health)
        if _current_health <= 0:
            died.emit()

func _init() -> void:
    _current_health = max_health

func take_damage(amount: int) -> void:
    current_health -= amount

func heal(amount: int) -> void:
    current_health += amount
```

### How It Works

```gdscript
# Create ONE instance accessed by multiple nodes:

# player.gd
@export var health_data: SharedHealthData

func _on_hit(damage: int) -> void:
    health_data.take_damage(damage)

# health_bar_ui.gd (COMPLETELY DIFFERENT SCENE)
@export var health_data: SharedHealthData

func _ready() -> void:
    health_data.health_changed.connect(_update_bar)

func _update_bar(current: int, maximum: int) -> void:
    value = float(current) / maximum

# enemy_ai.gd (YET ANOTHER SCENE)
@export var player_health: SharedHealthData

func _process(delta: float) -> void:
    if player_health.current_health < 20:
        # Player is weak — attack more aggressively!
        aggression = 2.0
```

### Why This Changes Everything

```
WITHOUT Resource Bus:
Player → signal → HUD → signal → HealthBar  (3 hops)
Player → signal → EnemyManager → signal → Enemy  (3 hops)

WITH Resource Bus:
Player writes → SharedHealthData ← HealthBar reads
                                 ← Enemy reads
(Zero signals between scenes, zero coupling, zero cascading)
```

> **⚠️ When to use:** Cross-scene data sharing (UI reading game state, AI reading player state). **When NOT to use:** Same-scene sibling communication — prefer the Root Mediator pattern instead.

---

## Composition vs Inheritance

> The community agrees: **prefer composition**, but **don't avoid inheritance entirely**. Use each where it shines.

### Use Composition For:

```
✅ Behaviors that could apply to MANY different entities
   (HealthComponent, HitboxComponent, MovementComponent)
✅ Features that might be added/removed at runtime
   (BuffComponent, ShieldComponent)
✅ Things that change independently of the core entity
   (AI behavior, visual effects, audio)
```

### Use Inheritance For:

```
✅ Broad shared attributes across a TYPE
   (Enemy base class → Goblin, Skeleton, Dragon)
✅ When polymorphism genuinely helps
   (Abstract Weapon → Sword.attack(), Bow.attack(), Staff.attack())
✅ When it's a SHALLOW hierarchy (1-2 levels deep, never more)
   (BasePickup → HealthPickup, ManaPickup)
```

### The Hybrid Approach (Community Consensus)

```gdscript
# INHERITANCE for the "what it IS" (shallow)
# enemy_base.gd
extends CharacterBody2D
class_name EnemyBase

@export var data: EnemyData
@onready var health: HealthComponent = $HealthComponent  # composition
@onready var hitbox: HitboxComponent = $HitboxComponent  # composition

func _ready() -> void:
    health.max_health = data.max_health
    hitbox.hit_received.connect(_on_hit)

func _on_hit(damage: int, source: Node) -> void:
    health.take_damage(damage)

# COMPOSITION for the "what it DOES" (flexible)
# goblin.gd — Inherits base, composes behavior
extends EnemyBase

@onready var patrol: PatrolComponent = $PatrolComponent  # unique to goblin
# Slime wouldn't have patrol, it would have BounceComponent instead
```

```
Rule of thumb:
- 1 level of inheritance = Almost always fine
- 2 levels = Proceed with caution
- 3+ levels = Almost always wrong in Godot. Refactor to composition.
```

---

## Atomic Design for Godot

Borrow from Brad Frost's Atomic Design — organize components by complexity:

```
res://components/
├── atoms/                    ← Smallest, most reusable
│   ├── health_bar.tscn       (ProgressBar + script)
│   ├── damage_number.tscn    (Label + tween)
│   ├── flash_component.tscn  (Node + shader)
│   └── timer_component.tscn  (Timer + signal wrapper)
│
├── molecules/                ← Combine atoms
│   ├── hitbox.tscn           (Area2D + CollisionShape)
│   ├── hurtbox.tscn          (Area2D + CollisionShape + signal)
│   ├── health_component.tscn (Node + HealthBar atom)
│   └── inventory_slot.tscn   (Panel + TextureRect + Label)
│
├── organisms/                ← Full features
│   ├── player_controller.tscn (CharacterBody + many molecules)
│   ├── enemy_ai.tscn          (CharacterBody + StateMachine + nav)
│   ├── inventory_panel.tscn   (Panel + GridContainer + slots)
│   └── dialog_box.tscn        (Panel + RichTextLabel + typewriter)
│
└── templates/                ← Page-level layouts
    ├── hud_layout.tscn        (CanvasLayer + organism arrangement)
    ├── level_template.tscn    (Node2D + spawn points + triggers)
    └── menu_layout.tscn       (Control + container structure)
```

### Why This Structure Works for AI

An AI agent looking at `components/atoms/health_bar.tscn` immediately knows:
1. It's **small** and **self-contained** (atom)
2. It does **one thing** (health bar)
3. It can be **dropped into any scene**
4. Its inputs are `@export` variables
5. Its outputs are signals

No context needed. No 500-line file to parse. No hidden dependencies.

---

## The Component Contract

Every component script should follow this template:

```gdscript
## A brief one-line description of what this component does.
##
## Detailed explanation of behavior, dependencies, and usage.
## This documentation shows in Godot's built-in help.
extends Node2D  # or whatever base type
class_name HealthComponent  # Always provide a class_name

# ─── SIGNALS (outputs — what this component tells the world) ───

signal health_changed(current: int, maximum: int)
signal died
signal healed(amount: int)

# ─── EXPORTS (inputs — what the parent configures) ───

@export_group("Configuration")
@export var max_health: int = 100
@export var invincibility_duration: float = 0.5
@export var show_damage_numbers: bool = true

@export_group("References")
## Optional: connect to a HealthBar to auto-update it
@export var health_bar: ProgressBar

# ─── PRIVATE STATE ───

var _health: int
var _is_invincible: bool = false

# ─── LIFECYCLE ───

func _ready() -> void:
    _health = max_health
    if health_bar:
        health_bar.max_value = max_health
        health_bar.value = max_health

# ─── PUBLIC API (methods the parent calls DOWN to) ───

## Deal damage to this entity. Returns actual damage dealt.
func take_damage(amount: int, source: Node = null) -> int:
    if _is_invincible or _health <= 0:
        return 0
    
    var actual := mini(amount, _health)
    _health -= actual
    health_changed.emit(_health, max_health)
    
    if health_bar:
        health_bar.value = _health
    
    if _health <= 0:
        died.emit()
    else:
        _start_invincibility()
    
    return actual

## Heal this entity. Returns actual amount healed.
func heal(amount: int) -> int:
    var actual := mini(amount, max_health - _health)
    _health += actual
    health_changed.emit(_health, max_health)
    healed.emit(actual)
    
    if health_bar:
        health_bar.value = _health
    
    return actual

## Get current health.
func get_health() -> int:
    return _health

## Check if dead.
func is_dead() -> bool:
    return _health <= 0

# ─── PRIVATE METHODS ───

func _start_invincibility() -> void:
    _is_invincible = true
    await get_tree().create_timer(invincibility_duration).timeout
    _is_invincible = false
```

### What Makes This AI-Friendly:

1. **Documentation comments** (`##`) — AI reads these for context, and they show in Godot's built-in help
2. **Grouped exports** — Clear what needs configuration
3. **Typed signals** — AI knows what data flows out
4. **Public API section** — Clear contract, documented return values
5. **Private variables** — Prefixed with `_`, signals not to touch
6. **No external dependencies** — Doesn't reach up the tree
7. **Optional references** — `health_bar` is optional, not required

---

## Self-Contained Scene Pattern

Each component scene is a **complete, testable unit**:

```
health_component.tscn
├── HealthComponent (Node) [health_component.gd]
│   └── InvincibilityTimer (Timer) — internal only

hitbox.tscn
├── HitBox (Area2D) [hitbox.gd]
│   └── CollisionShape2D

velocity_component.tscn
├── VelocityComponent (Node) [velocity_component.gd]
```

### The Litmus Test

> **Can this scene be opened alone and not throw errors?**
> If yes → Well-designed component. ✅
> If no → It has hidden dependencies that need fixing. ❌

---

## Vertical Slice Folder Structure

> The Godot community strongly recommends **organizing by feature/entity**, not by file type. This is called "vertical slicing."

### ❌ DON'T: Organize by Type (Horizontal)

```
res://
├── scripts/        ← All scripts dumped here
│   ├── player.gd
│   ├── enemy.gd
│   └── bullet.gd
├── scenes/         ← All scenes dumped here
│   ├── player.tscn
│   ├── enemy.tscn
│   └── bullet.tscn
├── sprites/        ← All sprites dumped here
│   ├── player.png
│   ├── enemy.png
│   └── bullet.png
└── sounds/
    ├── player_jump.wav
    └── enemy_death.wav

# Problem: To work on "Player" you need 4+ different folders open
# Problem: To delete "Bullet" you must hunt through every folder
# Problem: AI must search everywhere to understand one entity
```

### ✅ DO: Organize by Entity/Feature (Vertical)

```
res://
├── entities/
│   ├── player/
│   │   ├── player.tscn
│   │   ├── player.gd
│   │   ├── player_sprite.png
│   │   └── player_jump.wav
│   ├── enemies/
│   │   ├── goblin/
│   │   │   ├── goblin.tscn
│   │   │   ├── goblin.gd
│   │   │   └── goblin_sprite.png
│   │   └── slime/
│   │       ├── slime.tscn
│   │       └── slime.gd
│   └── pickups/
│       ├── coin/
│       └── health_potion/
│
├── systems/
│   ├── combat/
│   ├── inventory/
│   ├── dialog/
│   └── save/
│
├── shared/                     ← Cross-entity resources
│   ├── components/             ← Reusable components
│   ├── shaders/
│   ├── themes/
│   └── fonts/
│
├── ui/
│   ├── hud/
│   ├── menus/
│   └── dialogs/
│
├── levels/
│   ├── level_01/
│   └── level_02/
│
└── autoloads/
    ├── game_manager.gd
    ├── audio_manager.gd
    └── event_bus.gd
```

### Why Vertical Wins

```
1. To work on "Goblin": open ONE folder
2. To delete "Goblin": delete ONE folder
3. AI reads ONE folder to understand ONE entity
4. No merge conflicts when team members work on different entities
5. Aligns with Godot's scene-as-component philosophy
```

---

## The Event Bus

> For truly global events where distant nodes need to communicate across the entire scene tree. **Use sparingly** — most communication should be local (Call Down, Signal Up).

```gdscript
# event_bus.gd (Autoload: "Events")
extends Node

# ─── Game state ───
signal game_started
signal game_paused
signal game_over(score: int)

# ─── Player events ───
signal player_died
signal player_respawned
signal player_health_changed(current: int, maximum: int)

# ─── Level events ───
signal level_loaded(level_name: String)
signal checkpoint_reached(checkpoint_id: String)

# ─── Economy ───
signal coins_changed(amount: int)
signal item_purchased(item_id: String)

# ─── Combat ───
signal enemy_killed(enemy_type: String, position: Vector2)
signal boss_defeated(boss_name: String)
```

```gdscript
# EMITTING from anywhere:
Events.enemy_killed.emit("goblin", global_position)

# LISTENING from anywhere:
func _ready() -> void:
    Events.enemy_killed.connect(_on_enemy_killed)

func _on_enemy_killed(type: String, pos: Vector2) -> void:
    score += enemy_scores[type]
    spawn_particles(pos)
```

### Event Bus Rules

```
USE the Event Bus for:
✅ Truly global events (game over, level loaded, pause)
✅ UI reacting to gameplay (score changed, health changed)
✅ Achievement/analytics tracking
✅ Things that 3+ unrelated systems need to know about

DON'T use the Event Bus for:
❌ Communication between parent and child (use direct signals)
❌ Communication between siblings (use root mediator)
❌ Communication within one entity (keep it local)
❌ Everything (makes debugging a nightmare)
```

---

## Dependency Injection

### Three Flavors in Godot

```gdscript
# ─── 1. @export Injection (Most Common, Best for AI) ───
# Set in the editor Inspector or by parent script
@export var target: Node2D
@export var weapon_data: WeaponResource
@export var health_bar: ProgressBar

# ─── 2. Method Injection ───
# Parent passes dependency to child after instantiation
func setup(player_ref: CharacterBody2D, config: GameConfig) -> void:
    player = player_ref
    speed = config.enemy_speed

# ─── 3. Group-based Discovery (Loose) ───
# Find dependencies at runtime without hardcoded paths
func _ready() -> void:
    await get_tree().process_frame  # Wait for tree to settle
    var players := get_tree().get_nodes_in_group("player")
    if not players.is_empty():
        target = players[0]
```

### ❌ Anti-Pattern: Reaching

```gdscript
# NEVER do this — it creates invisible, breakable dependencies
func _ready() -> void:
    health_bar = get_node("../../UI/HUD/HealthBar")  # Fragile!
    player = get_parent().get_parent().get_node("Player")  # Suicidal!
    game_manager = get_node("/root/Main/Systems/GameManager")  # Explosive!
```

---

## Callable Injection

> A powerful Godot 4 pattern using `Callable` to inject behavior without knowing the implementation. Think of it as passing functions as props (like React callbacks on steroids).

```gdscript
# movement_component.gd
extends Node
class_name MovementComponent

## Inject custom movement logic without inheritance
@export var speed: float = 100.0
var _get_direction: Callable  # Injected behavior

func setup_direction_source(callable: Callable) -> void:
    _get_direction = callable

func _physics_process(delta: float) -> void:
    if _get_direction.is_valid():
        var direction: Vector2 = _get_direction.call()
        var body := get_parent() as CharacterBody2D
        if body:
            body.velocity = direction * speed
            body.move_and_slide()
```

```gdscript
# player.gd — Injects "get direction from input"
func _ready() -> void:
    $MovementComponent.setup_direction_source(
        func() -> Vector2:
            return Input.get_vector("left", "right", "up", "down")
    )

# enemy.gd — Injects "get direction from AI navigation"
func _ready() -> void:
    $MovementComponent.setup_direction_source(
        func() -> Vector2:
            return global_position.direction_to(nav_agent.get_next_path_position())
    )

# npc.gd — Injects "get direction from patrol path"
func _ready() -> void:
    $MovementComponent.setup_direction_source(
        func() -> Vector2:
            return global_position.direction_to(patrol_points[current_point].global_position)
    )
```

> **Same MovementComponent. Three totally different behaviors. Zero inheritance.**

---

## Data-Driven Architecture

**Separate data from behavior.** AI agents excel at creating data files.

```gdscript
# enemy_data.gd (Resource definition)
extends Resource
class_name EnemyData

@export var display_name: String = ""
@export var max_health: int = 30
@export var speed: float = 50.0
@export var damage: int = 5
@export var attack_range: float = 32.0
@export var detection_range: float = 150.0
@export var score_value: int = 100
@export var loot_table: Array[LootEntry] = []
@export var sprite_frames: SpriteFrames
@export_multiline var description: String = ""
```

```gdscript
# enemy_base.gd — ONE script drives ALL enemy types
extends CharacterBody2D

@export var data: EnemyData  # Just swap the data resource!

@onready var health: HealthComponent = $HealthComponent
@onready var sprite: AnimatedSprite2D = $AnimatedSprite2D

func _ready() -> void:
    health.max_health = data.max_health
    sprite.sprite_frames = data.sprite_frames
```

Now creating a new enemy type is **just creating a .tres file** — no code changes needed:

```
data/enemies/
├── goblin.tres      → display_name: "Goblin", health: 30, speed: 60
├── skeleton.tres    → display_name: "Skeleton", health: 50, speed: 40
├── dragon.tres      → display_name: "Dragon", health: 500, speed: 30
└── slime.tres       → display_name: "Slime", health: 15, speed: 80
```

An AI agent can create dozens of enemy variants by just generating `.tres` files.

---

## The Registry Pattern

For when components need to find each other without coupling:

```gdscript
# component_registry.gd (Autoload)
extends Node

var _registries: Dictionary = {}

func register(category: String, component: Node) -> void:
    if not _registries.has(category):
        _registries[category] = []
    _registries[category].append(component)
    component.tree_exiting.connect(func(): unregister(category, component))

func unregister(category: String, component: Node) -> void:
    if _registries.has(category):
        _registries[category].erase(component)

func get_all(category: String) -> Array[Node]:
    return _registries.get(category, [])

func get_nearest(category: String, position: Vector2) -> Node:
    var components := get_all(category)
    if components.is_empty():
        return null
    var nearest: Node = null
    var nearest_dist := INF
    for comp in components:
        if comp is Node2D:
            var dist := position.distance_squared_to(comp.global_position)
            if dist < nearest_dist:
                nearest_dist = dist
                nearest = comp
    return nearest
```

Usage — components self-register:
```gdscript
# enemy.gd
func _ready() -> void:
    ComponentRegistry.register("enemy", self)
    ComponentRegistry.register("targetable", self)

# player_targeting.gd — find nearest WITHOUT knowing where enemies are
func find_target() -> Node:
    return ComponentRegistry.get_nearest("targetable", global_position)
```

---

## Unit Testing with GUT

> **GUT** (Godot Unit Test) is the community-standard testing framework. Modular components should be tested.

### Setup

```
1. Install GUT from Asset Library (or https://github.com/bitwes/Gut)
2. Enable the plugin: Project Settings → Plugins → GUT → ✅
3. Create test folder: res://test/
4. Create test scenes/scripts prefixed with test_
```

### Testing a Component

```gdscript
# test/test_health_component.gd
extends GutTest

var health_comp: HealthComponent

func before_each() -> void:
    health_comp = HealthComponent.new()
    health_comp.max_health = 100
    add_child_autofree(health_comp)  # Auto-cleanup

func test_initial_health() -> void:
    assert_eq(health_comp.get_health(), 100, "Should start at max health")

func test_take_damage() -> void:
    var dealt := health_comp.take_damage(30)
    assert_eq(dealt, 30, "Should deal 30 damage")
    assert_eq(health_comp.get_health(), 70, "Health should be 70")

func test_take_lethal_damage() -> void:
    watch_signals(health_comp)
    health_comp.take_damage(150)
    assert_signal_emitted(health_comp, "died", "Should emit died signal")
    assert_true(health_comp.is_dead(), "Should be dead")

func test_heal_does_not_exceed_max() -> void:
    health_comp.take_damage(30)
    var healed := health_comp.heal(50)  # Only 30 missing
    assert_eq(healed, 30, "Should only heal 30")
    assert_eq(health_comp.get_health(), 100, "Should cap at max")

func test_invincibility_blocks_damage() -> void:
    health_comp.take_damage(10)  # Triggers invincibility
    var dealt := health_comp.take_damage(10)  # Should be blocked
    assert_eq(dealt, 0, "Should block during invincibility")
```

### Why Testing Matters for Modularity

```
1. Components that pass tests can be safely modified by AI
2. New features get regression tests automatically
3. Refactoring becomes fearless
4. AI can run tests to verify its own changes
```

---

## Cross-Project Plugins

> Make your best components reusable as Godot plugins you can drop into any project.

### Plugin Structure

```
your_project/
└── addons/
    └── health_system/
        ├── plugin.cfg
        ├── health_plugin.gd
        ├── components/
        │   ├── health_component.tscn
        │   ├── health_component.gd
        │   └── health_bar.tscn
        ├── resources/
        │   └── health_config.gd
        └── README.md
```

### plugin.cfg

```ini
[plugin]
name="Health System"
description="Modular health, damage, and healing components"
author="Your Name"
version="1.0.0"
script="health_plugin.gd"
```

### Plugin Script

```gdscript
# health_plugin.gd
@tool
extends EditorPlugin

func _enter_tree() -> void:
    # Register custom types so they appear in "Add Node" dialog
    add_custom_type(
        "HealthComponent",
        "Node",
        preload("components/health_component.gd"),
        preload("icon.png")
    )

func _exit_tree() -> void:
    remove_custom_type("HealthComponent")
```

### Reuse Across Projects

```
Option 1: Copy the addons/health_system/ folder
Option 2: Git submodule (git submodule add <repo_url> addons/health_system)
Option 3: Publish to Godot Asset Library
```

---

## File Conventions for AI

### Naming That Tells the Story

```
# An AI (or human) should understand the codebase from filenames alone:

entities/player/player.gd               → Player entity
entities/enemies/goblin/goblin.gd       → Goblin enemy
shared/components/health_component.gd   → Health system
shared/components/hitbox.gd             → Hit detection
systems/combat/damage_calculator.gd     → Damage math
systems/inventory/inventory_manager.gd  → Inventory logic
autoloads/event_bus.gd                  → Global signals
data/weapons/iron_sword.tres            → Iron sword stats
```

### Every Script Has a Header

```gdscript
## Handles player movement, jumping, and wall interactions.
##
## Dependencies: None (self-contained)
## Signals OUT: died, health_changed
## Exports IN: speed, jump_force, gravity_scale
## Groups: "player"
##
## Usage: Attach to a CharacterBody2D with CollisionShape2D child.
extends CharacterBody2D
class_name Player
```

An AI reading this header knows:
- What the script does
- What it depends on
- What signals it emits
- What it expects to be configured
- What group to find it in
- How to set it up

---

## Feature Module Boundaries

### Module Rules

```
1. Modules ONLY communicate through signals, Autoloads, or shared Resources
2. No module imports another module's internal scripts
3. Each module has a clear PUBLIC API (the module.gd file or exported signals)
4. Data definitions (Resources) can be shared via shared/ directory
5. A module can be DELETED without breaking other modules
```

### Test: Can You Delete It?

> The ultimate modularity test: **Can you delete an entire feature folder and have the game still compile?** (With reduced functionality, obviously.)
>
> If yes → Your boundaries are clean ✅
> If no → There are hidden cross-module dependencies ❌

---

## Practical Template

### Creating a New Component (AI Workflow)

When an AI is asked "add a stamina system", it should:

1. **Create the Resource:**
```gdscript
# shared/resources/stamina_config.gd
extends Resource
class_name StaminaConfig
@export var max_stamina: float = 100.0
@export var regen_rate: float = 15.0
@export var regen_delay: float = 1.0
```

2. **Create the Component:**
```gdscript
# shared/components/stamina_component.gd  +  .tscn
## Manages stamina with drain, regeneration, and exhaustion.
extends Node
class_name StaminaComponent

signal stamina_changed(current: float, maximum: float)
signal exhausted
signal recovered

@export var config: StaminaConfig
var _stamina: float
var _regen_timer: float = 0.0

func _ready() -> void:
    _stamina = config.max_stamina

func _process(delta: float) -> void:
    if _regen_timer > 0:
        _regen_timer -= delta
    elif _stamina < config.max_stamina:
        _stamina = minf(_stamina + config.regen_rate * delta, config.max_stamina)
        stamina_changed.emit(_stamina, config.max_stamina)
        if _stamina >= config.max_stamina:
            recovered.emit()

func use_stamina(amount: float) -> bool:
    if _stamina < amount:
        exhausted.emit()
        return false
    _stamina -= amount
    _regen_timer = config.regen_delay
    stamina_changed.emit(_stamina, config.max_stamina)
    return true

func get_stamina_percent() -> float:
    return _stamina / config.max_stamina
```

3. **Create a Test:**
```gdscript
# test/test_stamina_component.gd
extends GutTest

func test_use_stamina_returns_false_when_empty() -> void:
    var comp := StaminaComponent.new()
    comp.config = StaminaConfig.new()
    comp.config.max_stamina = 10.0
    add_child_autofree(comp)
    comp.use_stamina(10.0)
    assert_false(comp.use_stamina(1.0), "Should fail when empty")
```

4. **Wire into Player:**
```gdscript
# In player.tscn — add StaminaComponent as child
# In player.gd — reference it:
@onready var stamina: StaminaComponent = $StaminaComponent

func sprint(delta: float) -> void:
    if stamina.use_stamina(20.0 * delta):
        speed = run_speed
    else:
        speed = walk_speed
```

**Zero changes to existing code.** The component is additive.

---

## Anti-Patterns

### Things the Community Says NEVER Do

```
1. ❌ GOD SCRIPTS — 1000+ line scripts that do everything
   Fix: Split into components (HealthComponent, MovementComponent, etc.)

2. ❌ AUTOLOAD EVERYTHING — Putting all game logic in singletons
   Fix: Use autoloads ONLY for truly global state (EventBus, SaveManager)

3. ❌ get_parent() / get_node("../../") — Reaching up the tree
   Fix: Use @export injection or signals

4. ❌ DEEP INHERITANCE — 4+ levels of extends
   Fix: Flatten to 1-2 levels, use composition for the rest

5. ❌ UNNAMED COMPONENTS — Nodes with default names (Node2D, Area2D)
   Fix: ALWAYS name nodes descriptively (HitBox, PatrolPath, SpawnPoint)

6. ❌ MAGIC NUMBERS — Hardcoded values scattered through scripts
   Fix: Use @export or Resource files for ALL configurable values

7. ❌ PREMATURE ARCHITECTURE — Over-engineering before you need it
   Fix: Start simple, refactor when you hit pain points
   (Community wisdom: "Make it work, make it right, make it fast")

8. ❌ TYPE-BASED FOLDERS — Separating scripts/ scenes/ sprites/
   Fix: Use vertical slicing (entity-based folders)
```

---

## Modularity Checklist

Before committing any component, verify:

```
□ Has class_name
□ Has ## documentation header with Dependencies/Signals/Exports listed
□ Uses @export for ALL configuration (no magic numbers)
□ Uses signals for ALL outgoing/upward communication
□ Calls DOWN to children, never UP to parents
□ Does NOT use get_parent() or hardcoded paths
□ Can be instantiated in an empty scene without errors
□ Has no circular dependencies
□ Private methods/vars prefixed with _
□ File is <200 lines (split if larger)
□ One responsibility (doesn't do two unrelated things)
□ Has a unit test (or at minimum, a test scene)
□ Lives in the correct folder (entity/ or shared/components/)
□ Could be deleted without breaking the rest of the project
```

---

*← [18 — Project Structure](./18-project-structure.md) | [README](./README.md) →*
