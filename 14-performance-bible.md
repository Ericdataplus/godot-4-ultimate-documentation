# 14 — Performance Bible

> Profile first, optimize second. Every technique the community has found for keeping your game running smooth.

---

## Table of Contents

- [Golden Rules](#golden-rules)
- [Profiling Tools](#profiling-tools)
- [GDScript Optimization](#gdscript-optimization)
- [Rendering Optimization](#rendering-optimization)
- [Physics Optimization](#physics-optimization)
- [Alternative Physics Engines](#alternative-physics-engines)
- [Memory Management](#memory-management)
- [Loading & Streaming](#loading--streaming)
- [Mobile & Web Optimization](#mobile--web)
- [Common Performance Killers](#common-performance-killers)

---

## Golden Rules

```
1. PROFILE BEFORE OPTIMIZING — Don't guess, measure!
2. Optimize the BIGGEST bottleneck first
3. 80% of slowness comes from 20% of code
4. CPU-bound vs GPU-bound are DIFFERENT problems
5. Premature optimization is the root of all evil
   (But some patterns are just always better)
6. A SHIPPED game at 30fps beats a perfect game that never ships
```

---

## Profiling Tools

### Built-in Profiler

```
Debug → Profiler (while game is running)
- Shows function execution times
- Shows idle vs physics frame time
- Shows rendering time

Target: 16.6ms per frame = 60fps
        33.3ms per frame = 30fps

CPU vs GPU bound:
- If IDLE time is high → CPU-bound (optimize scripts/physics)
- If RENDER time is high → GPU-bound (optimize draw calls/shaders)
```

### Performance Monitors

```gdscript
# In code
var fps := Engine.get_frames_per_second()
var process_time := Performance.get_monitor(Performance.TIME_PROCESS)
var physics_time := Performance.get_monitor(Performance.TIME_PHYSICS_PROCESS)

# Draw calls
var draw_calls := Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME)

# Object count
var node_count := Performance.get_monitor(Performance.OBJECT_NODE_COUNT)
var resource_count := Performance.get_monitor(Performance.OBJECT_RESOURCE_COUNT)

# Memory
var static_memory := Performance.get_monitor(Performance.MEMORY_STATIC)
var video_memory := Performance.get_monitor(Performance.RENDER_VIDEO_MEM_USED)

# Physics
var physics_objects := Performance.get_monitor(Performance.PHYSICS_2D_ACTIVE_OBJECTS)
var collision_pairs := Performance.get_monitor(Performance.PHYSICS_2D_COLLISION_PAIRS)
```

### Debug Overlay

```gdscript
# Quick performance HUD — add to an Autoload or CanvasLayer
extends Label

func _process(delta: float) -> void:
    text = "FPS: %d\nDraw: %d\nNodes: %d\nPhysics: %.1fms\nMemory: %.1fMB" % [
        Engine.get_frames_per_second(),
        Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME),
        Performance.get_monitor(Performance.OBJECT_NODE_COUNT),
        Performance.get_monitor(Performance.TIME_PHYSICS_PROCESS) * 1000.0,
        OS.get_static_memory_usage() / 1048576.0,
    ]
```

---

## GDScript Optimization

### The Big Wins

```gdscript
# 1. STATIC TYPING (can be up to 2x faster for math-heavy code)
# ❌ var speed = 200
# ✅ var speed: float = 200.0

# 2. CACHE NODE REFERENCES
# ❌ $Sprite2D.rotation += delta (traverses tree every frame)
# ✅ @onready var sprite := $Sprite2D (cached once)

# 3. AVOID STRING OPERATIONS IN LOOPS
# Strings are immutable — concatenation creates new objects
# ❌ for i in 1000: result += str(i) + ", "
# ✅ var parts: PackedStringArray; parts.append(str(i)); ", ".join(parts)

# 4. USE distance_squared_to() FOR RANGE CHECKS
# ❌ pos.distance_to(target) < range (uses sqrt — expensive!)
# ✅ pos.distance_squared_to(target) < range * range

# 5. PRE-CALCULATE LOOP INVARIANTS
# ❌ for enemy in enemies: var d = cos(angle) * speed
# ✅ var cos_val := cos(angle); for enemy in enemies: var d = cos_val * speed

# 6. USE BUILT-IN FUNCTIONS (they're C++ under the hood)
# ❌ Manual lerp: a + (b - a) * t
# ✅ Built-in: lerp(a, b, t), move_toward(), direction_to()
# Also use: Geometry2D, PhysicsServer, SurfaceTool, MeshDataTool

# 7. AVOID CREATING OBJECTS IN _process
# ❌ func _process: var v = Vector2(x, y) (GC pressure)
# ✅ Reuse: velocity.x = x; velocity.y = y

# 8. USE PackedArrays FOR BULK DATA
# ❌ Array[float] (boxed, heap allocated)
# ✅ PackedFloat32Array (contiguous, fast)

# 9. MINIMIZE SIGNAL CONNECTIONS/DISCONNECTIONS
# Don't connect/disconnect every frame

# 10. CONSIDER C# OR GDEXTENSION FOR HOT PATHS
# GDScript is ~100x slower than C++ for math-heavy code
# That said, most games are NOT CPU-bound in GDScript
```

### Processing Control

```gdscript
# CRITICAL: Empty _process/_physics_process functions still cost CPU!
# If a script doesn't need per-frame logic, don't include the function at all.

# Don't process nodes that don't need it
func _ready() -> void:
    set_process(false)           # Disable _process
    set_physics_process(false)   # Disable _physics_process

# Enable only when needed
func activate() -> void:
    set_process(true)

# Process mode for pausing
process_mode = Node.PROCESS_MODE_PAUSABLE   # Pauses with tree
process_mode = Node.PROCESS_MODE_ALWAYS     # Never pauses
process_mode = Node.PROCESS_MODE_DISABLED   # Never processes
```

### Stagger Heavy Computation

```gdscript
# Don't update EVERYTHING every frame — spread across frames

var _frame_counter: int = 0
var _enemies: Array[Node] = []

func _physics_process(delta: float) -> void:
    _frame_counter += 1
    
    # Update 1/3 of enemies per frame (cycle through groups)
    for i in range(_enemies.size()):
        if i % 3 == _frame_counter % 3:
            _enemies[i].update_ai(delta * 3.0)
```

---

## Rendering Optimization

### Reduce Draw Calls

```
Draw calls = number of times the GPU draws something
Every unique material + mesh combination = 1 draw call
100+ draw calls = potential problem (mobile)
1000+ draw calls = definitely a problem

Solutions:
1. Share materials between objects (same material = batched)
2. Use MultiMeshInstance for repeated objects (trees, bullets, crowds)
3. Merge static meshes in the editor
4. Use texture atlases for 2D sprites
5. Reduce unique shaders
```

### MultiMesh — The Community's Secret Weapon

```gdscript
# Use MultiMeshInstance for ANYTHING you have hundreds of:
# grass, trees, bullets, crowd NPCs, particles, coins, debris

# Setup
var multi_mesh := MultiMesh.new()
multi_mesh.transform_format = MultiMesh.TRANSFORM_2D  # or TRANSFORM_3D
multi_mesh.instance_count = 1000
multi_mesh.mesh = your_mesh

# Set transforms for each instance
for i in 1000:
    multi_mesh.set_instance_transform_2d(i, Transform2D(0, random_pos))

# Dramatically reduces draw calls: 1000 objects → 1 draw call
```

### 2D Specific

```gdscript
# Visibility optimization
# Objects outside the viewport don't render, BUT they still process!
# Use VisibleOnScreenNotifier2D to disable off-screen logic

@onready var visibility: VisibleOnScreenNotifier2D = $VisibleOnScreenNotifier2D

func _ready() -> void:
    visibility.screen_entered.connect(func(): set_process(true))
    visibility.screen_exited.connect(func(): set_process(false))

# Or use VisibleOnScreenEnabler2D which does this automatically!
# Add as child → set enable_mode → picks up parent processing

# Y-sort for proper layering (cheaper than Z-index manipulation)
# Enable "Y Sort Enabled" on the parent node

# Tilemap optimization
# Use a single TileMapLayer instead of multiple for the same content
# Godot batches tiles in chunks automatically
```

### 3D Specific

```gdscript
# LOD (Level of Detail) — Godot 4.2+ auto-generates
# Meshes simplify at distance. Configure in import settings.
mesh_instance.lod_bias = 1.0  # Lower = more aggressive LOD

# Occlusion Culling — don't render what's behind walls
# 1. Enable: Project Settings → Rendering → Occlusion Culling → Use Occlusion Culling
# 2. Add OccluderInstance3D nodes for large walls/buildings
# 3. Bake occlusion data

# Camera Far Plane — reduce render distance
camera.far = 500.0  # Combine with distance fog so it's not obvious

# VisibleOnScreenNotifier3D — same as 2D version but for 3D
# Auto-enable/disable processing based on camera visibility

# Shadow Distance
# Reduce directional_shadow_max_distance for better shadow quality nearby

# Lightmap GI — Bake lighting for STATIC objects
# Dramatically faster than real-time GI (VoxelGI/SDFGI)
# 1. Set lights to bake mode: "Static"
# 2. Add LightmapGI node
# 3. UV unwrap meshes for lightmaps
# 4. Bake
```

### Label/Text Performance (Community-Discovered)

```
⚠️ KNOWN ISSUE (Godot 4.3/4.4):
   Label nodes with shadows and outlines are VERY expensive!
   A few Labels with shadow enabled can tank your framerate.

Workarounds:
1. Disable Label shadows unless absolutely needed
2. Use a shader-based text shadow instead
3. Limit the number of visible Labels at any time
4. Fix expected in Godot 4.5+
```

---

## Physics Optimization

```gdscript
# 1. REDUCE COLLISION SHAPES
# Simpler shapes = faster collision detection
# Circle > Rectangle > Capsule > Polygon (convex) >>> Polygon (concave)
# NEVER use concave polygon for moving objects

# 2. USE COLLISION LAYERS & MASKS
# Don't collide everything with everything
# Separate layers: Player, Enemies, Terrain, Projectiles, Pickups
# Only enable masks that actually need to interact

# 3. REDUCE PHYSICS TICK RATE
# Project Settings → Physics → Common → Physics Ticks Per Second
# 60 (default) → 30 for most games is fine
# Enable physics interpolation to smooth it out (Godot 4.3+)
# Set physics_jitter_fix = 0 when using interpolation

# 4. PHYSICS INTERPOLATION (Godot 4.3+)
# Project Settings → Physics → Common → Physics Interpolation → true
# Smooths rendering between physics ticks
# Lets you run physics at 30hz while rendering at 60/144fps
# This is a FREE performance win!

# 5. DISABLE PHYSICS FOR OFF-SCREEN OBJECTS
func _on_screen_exited() -> void:
    set_physics_process(false)
    $CollisionShape2D.set_deferred("disabled", true)

# 6. USE AREAS INSTEAD OF RAYCASTS
# For detection zones, Area2D/3D is cheaper than many raycasts

# 7. AVOID move_and_slide ON HUNDREDS OF OBJECTS
# For large crowds, use simpler movement:
# - Direct position manipulation
# - Grid-based sector checking (only check neighbors in same grid cell)
# - Custom steering/avoidance instead of physics

# 8. RUN PHYSICS ON SEPARATE THREAD
# Project Settings → Physics → Common → Run on Separate Thread → true
# Offloads physics from the main thread
# ⚠️ Still experimental — test thoroughly

# 9. FEWER SOLVER ITERATIONS
# Project Settings → Physics → 2D/3D → Solver Iterations
# Lower = faster but less accurate collision resolution
# Default 16 is fine for most games, try 8 for better perf
```

---

## Alternative Physics Engines

> **Community consensus:** Switch to alternative physics engines for significant performance gains.

### Jolt Physics (3D)

```
WHAT: AAA-grade physics engine (used in Horizon Forbidden West)
WHY:  130%+ better performance than Godot's default 3D physics
      More stable, accurate, and multi-core friendly
HOW:  Native in Godot 4.4+ (no plugin needed!)
      Project Settings → Physics → 3D → Physics Engine → "Jolt Physics"

Limitations:
• 3D only (no 2D support)
• Some Godot joint properties behave differently
• WorldBoundaryShape3D has smaller effective size
```

### Rapier 2D

```
WHAT: High-performance 2D physics engine written in Rust
WHY:  Significantly faster for 2D-heavy games
HOW:  Install from Asset Library: "Godot Rapier 2D"
      Project Settings → Physics → 2D → Physics Engine → "Rapier2D"

Best for:
• Games with hundreds of 2D physics bodies
• Complex 2D collision scenarios
• When default 2D physics is your bottleneck
```

---

## Memory Management

```gdscript
# 1. FREE NODES PROPERLY
node.queue_free()  # Safe — waits until end of frame
# Don't just remove from tree, actually free the memory

# 2. PRELOAD SMALL ASSETS, LOAD LARGE ONES
# Preload (compile time, blocks startup):
const BULLET := preload("res://scenes/bullet.tscn")

# Load (runtime, can be threaded):
var level := load("res://scenes/level_big.tscn")

# 3. USE OBJECT POOLS (see 02-scene-architecture.md)
# Don't instantiate/free frequently-used objects every frame
# Pre-create a pool and recycle them

# 4. MONITOR MEMORY
print("Memory: %.1f MB" % (OS.get_static_memory_usage() / 1048576.0))

# 5. WEAK REFERENCES
# GDScript doesn't have weak refs, but you can check validity:
var target_id: int = target.get_instance_id()
# Check if still valid:
if is_instance_id_valid(target_id):
    var target_ref: Node = instance_from_id(target_id)

# 6. RESOURCE DEDUPLICATION
# Loading the same resource twice returns the SAME instance
# This is usually wanted! But watch out for shared state.
# Use resource.duplicate() when you need independent copies.

# 7. COMPRESS TEXTURES
# Import dock → Compression → VRAM compression for 3D
# Use smaller textures where possible (1024x1024 vs 4096x4096)
# Compress audio: OGG for music, WAV for short SFX

# 8. UNLOAD UNUSED RESOURCES
# Godot auto-unloads unreferenced resources
# But if you cache in dictionaries/arrays, clear them manually
```

---

## Loading & Streaming

### Background Loading

```gdscript
# Load heavy scenes without freezing the game
func load_level_async(path: String) -> void:
    ResourceLoader.load_threaded_request(path)
    
    $LoadingScreen.visible = true
    
    while true:
        var progress: Array = []
        var status := ResourceLoader.load_threaded_get_status(path, progress)
        match status:
            ResourceLoader.THREAD_LOAD_IN_PROGRESS:
                $LoadingScreen/ProgressBar.value = progress[0] * 100
            ResourceLoader.THREAD_LOAD_LOADED:
                var scene := ResourceLoader.load_threaded_get(path) as PackedScene
                get_tree().change_scene_to_packed(scene)
                return
            ResourceLoader.THREAD_LOAD_FAILED:
                push_error("Failed to load: " + path)
                return
        
        await get_tree().process_frame
```

### Lazy Loading Pattern

```gdscript
# Only load resources when they're actually needed
var _weapon_cache: Dictionary = {}

func get_weapon_resource(weapon_id: String) -> WeaponResource:
    if not _weapon_cache.has(weapon_id):
        _weapon_cache[weapon_id] = load("res://data/weapons/%s.tres" % weapon_id)
    return _weapon_cache[weapon_id]

# Clear cache when changing levels
func _on_level_changed() -> void:
    _weapon_cache.clear()
```

---

## Mobile & Web

```
MOBILE OPTIMIZATION:
├── Use GL Compatibility renderer (not Forward+)
├── Reduce texture sizes (1024x1024 max)
├── Limit particle counts (50-100 vs 500+)
├── Disable expensive post-processing (bloom, SSR, SSAO)
├── Target 30fps if needed (use physics interpolation)
├── Batch 2D draw calls via texture atlases
├── Minimize shader complexity
└── Test on actual devices, not just emulators

WEB OPTIMIZATION:
├── Use GL Compatibility renderer
├── Minimize total download size
├── Defer loading non-essential content
├── Consider disabling threads for broader browser support
├── Audio starts only after user interaction (browser policy)
└── File I/O is limited — use user:// for saves
```

---

## Common Performance Killers

> Things the community has repeatedly found kill performance:

```
1. ❌ EMPTY _process() FUNCTIONS
   Even an empty _process() has overhead per frame per node.
   Don't include it unless you need it!

2. ❌ LABEL SHADOWS (Godot 4.3/4.4)
   Known bug — Label with shadows/outlines is very expensive.

3. ❌ HUNDREDS OF AREA2D OVERLAP CHECKS
   Each Area checking overlaps with every other Area = O(n²).
   Use collision layers to reduce pairs.

4. ❌ get_node() IN LOOPS
   Cache references with @onready, don't traverse per-frame.

5. ❌ DYNAMIC FONTS WITH MANY SIZES
   Each font size generates a new atlas. Stick to 2-3 sizes.

6. ❌ TOO MANY UNIQUE MATERIALS
   Each unique material breaks batching. Share materials!

7. ❌ INSTANTIATE/QUEUE_FREE IN _process
   Object pooling is dramatically faster for frequent spawning.

8. ❌ UNOPTIMIZED SHADERS ON EVERY PIXEL
   Complex fragment shaders on full-screen quads kill GPUs.
   Move calculations to vertex shader when possible.

9. ❌ UNCOMPRESSED TEXTURES
   A 4K uncompressed texture uses 64MB+ of VRAM.
   Enable VRAM compression in import settings.

10. ❌ CALLING print() IN _process
    String formatting every frame is surprisingly expensive.
    Remove debug prints before shipping!
```

---

*← [13 — GDExtension](./13-gdextension.md) | [15 — Editor Secrets](./15-editor-secrets.md) →*
