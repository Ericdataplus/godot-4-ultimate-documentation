# 08 — 3D Rendering & Environment

> Lighting, PBR materials, global illumination, sky, fog, and optimization.

---

## Table of Contents

- [3D Scene Setup](#3d-scene-setup)
- [Lighting](#lighting)
- [PBR Materials](#pbr-materials)
- [Global Illumination](#global-illumination)
- [Environment & Sky](#environment--sky)
- [Post-Processing](#post-processing)
- [3D Optimization](#3d-optimization)

---

## 3D Scene Setup

Every 3D scene needs at minimum:

```
Node3D (root)
├── Camera3D                    — The viewer
├── DirectionalLight3D          — Sun/moon
├── WorldEnvironment            — Sky, ambient, fog, effects
│   └── Environment resource
└── Your game content...
    ├── MeshInstance3D (ground)
    ├── MeshInstance3D (objects)
    └── CharacterBody3D (player)
```

### Camera3D Basics

```gdscript
@onready var camera: Camera3D = $Camera3D

func _ready() -> void:
    camera.fov = 75.0              # Field of view (60-90 typical)
    camera.near = 0.05             # Near clipping plane
    camera.far = 1000.0            # Far clipping plane
    camera.projection = Camera3D.PROJECTION_PERSPECTIVE
```

---

## Lighting

### Light Types

```gdscript
# DirectionalLight3D — Sun/moon (parallel rays, infinite range)
# Affects entire scene. One is usually enough.
directional_light.rotation_degrees = Vector3(-45, -30, 0)  # Angle
directional_light.light_energy = 1.0
directional_light.light_color = Color(1.0, 0.95, 0.9)     # Warm sunlight
directional_light.shadow_enabled = true

# OmniLight3D — Point light (emits in all directions)
omni_light.light_energy = 2.0
omni_light.omni_range = 10.0         # How far it reaches
omni_light.omni_attenuation = 1.0    # Falloff curve
omni_light.shadow_enabled = true

# SpotLight3D — Cone light (flashlight, car headlights)
spot_light.light_energy = 3.0
spot_light.spot_range = 15.0
spot_light.spot_angle = 30.0         # Cone width (degrees)
spot_light.spot_attenuation = 1.0
spot_light.shadow_enabled = true
```

### Shadow Quality

```gdscript
# Project Settings → Rendering → Lights and Shadows
# Directional Shadow Size: 4096 (quality) or 2048 (performance)
# Shadow filter: PCF13 (quality) or PCF5 (performance)

# Per-light shadow settings
directional_light.directional_shadow_mode = DirectionalLight3D.SHADOW_PARALLEL_4_SPLITS
directional_light.directional_shadow_max_distance = 100.0
```

### Baked vs Real-Time Lighting

```
Real-Time (Default):
├── Fully dynamic, objects cast proper shadows
├── More GPU intensive
└── Use for: moving objects, characters, dynamic scenes

Baked (LightmapGI):
├── Pre-calculated, stored in textures
├── Very fast at runtime
├── Only works on STATIC geometry
└── Use for: architectural interiors, static levels
```

---

## PBR Materials

### StandardMaterial3D

```gdscript
var mat := StandardMaterial3D.new()

# Albedo (base color)
mat.albedo_color = Color(0.8, 0.2, 0.2)   # Red
mat.albedo_texture = preload("res://textures/brick_albedo.png")

# Metallic
mat.metallic = 0.0          # 0 = non-metal (plastic, wood, skin)
mat.metallic_texture = null  # 1 = full metal (steel, gold)

# Roughness
mat.roughness = 0.8          # 0 = mirror smooth, 1 = fully rough
mat.roughness_texture = null

# Normal map (surface detail without geometry)
mat.normal_enabled = true
mat.normal_texture = preload("res://textures/brick_normal.png")
mat.normal_scale = 1.0

# Emission (self-illumination / glow)
mat.emission_enabled = true
mat.emission = Color(0.0, 1.0, 0.5)
mat.emission_energy_multiplier = 2.0

# Ambient Occlusion
mat.ao_enabled = true
mat.ao_texture = preload("res://textures/brick_ao.png")

# Transparency
mat.transparency = BaseMaterial3D.TRANSPARENCY_ALPHA
mat.albedo_color.a = 0.5
```

### Common Material Presets

```
Metal (steel):     metallic=0.9, roughness=0.3, albedo=gray
Gold:              metallic=1.0, roughness=0.2, albedo=gold
Wood:              metallic=0.0, roughness=0.7, albedo=brown
Plastic:           metallic=0.0, roughness=0.4, albedo=any
Glass:             metallic=0.0, roughness=0.0, transparency=alpha
Skin:              metallic=0.0, roughness=0.6, subsurface=on
Fabric:            metallic=0.0, roughness=0.9, albedo=any
Water:             metallic=0.0, roughness=0.0, transparency=alpha
```

---

## Global Illumination

### VoxelGI (Real-Time Indoor GI)

```
Best for: Indoor scenes, medium-sized environments
Pros: Real-time, dynamic lights work
Cons: Limited range, higher GPU cost

Setup:
1. Add VoxelGI node
2. Set its size to cover your scene
3. Click "Bake GI" in toolbar
4. Objects must have "Global Illumination → Static" or "Dynamic"
```

### SDFGI (Open-World GI)

```
Best for: Large outdoor scenes, open worlds  
Pros: Infinite range, no baking needed
Cons: Only one directional light, less precise

Setup:
1. In WorldEnvironment → Environment → SDFGI
2. Enable SDFGI
3. Adjust cascades and energy
4. No baking required!
```

### LightmapGI (Baked, Highest Quality)

```
Best for: Static architectural scenes (interiors)
Pros: Best quality, lowest runtime cost
Cons: Static only, requires UV2, long bake times

Setup:
1. All static meshes need UV2 (lightmap UVs)
2. Add LightmapGI node
3. Set quality and bounces
4. Click "Bake Lightmaps"
```

---

## Environment & Sky

### WorldEnvironment Setup

```gdscript
# Create environment resource
var env := Environment.new()

# Sky
var sky := Sky.new()
var sky_material := ProceduralSkyMaterial.new()
sky_material.sky_top_color = Color(0.35, 0.55, 0.9)
sky_material.sky_horizon_color = Color(0.65, 0.75, 0.9)
sky_material.ground_bottom_color = Color(0.15, 0.1, 0.1)
sky_material.ground_horizon_color = Color(0.65, 0.6, 0.55)
sky.sky_material = sky_material
env.sky = sky
env.background_mode = Environment.BG_SKY

# Ambient light
env.ambient_light_source = Environment.AMBIENT_SOURCE_SKY
env.ambient_light_energy = 0.5

# Fog
env.fog_enabled = true
env.fog_light_color = Color(0.65, 0.7, 0.8)
env.fog_density = 0.005
env.fog_sky_affect = 0.5

# Volumetric fog (more realistic but costly)
env.volumetric_fog_enabled = true
env.volumetric_fog_density = 0.02
env.volumetric_fog_emission = Color(0.5, 0.5, 0.5)

# Tonemap
env.tonemap_mode = Environment.TONE_MAP_FILMIC
env.tonemap_exposure = 1.0

# SSAO (Screen-Space Ambient Occlusion)
env.ssao_enabled = true
env.ssao_radius = 1.0
env.ssao_intensity = 2.0

# SSR (Screen-Space Reflections)
env.ssr_enabled = true
env.ssr_max_steps = 64

# Glow
env.glow_enabled = true
env.glow_intensity = 0.8
env.glow_bloom = 0.2
```

### Day/Night Cycle

```gdscript
@export var day_duration: float = 120.0  # Seconds for full cycle
@onready var sun: DirectionalLight3D = $DirectionalLight3D
@onready var world_env: WorldEnvironment = $WorldEnvironment

var time_of_day: float = 0.0  # 0-1 (0=midnight, 0.5=noon)

func _process(delta: float) -> void:
    time_of_day = fmod(time_of_day + delta / day_duration, 1.0)
    
    # Rotate sun
    var sun_angle := (time_of_day - 0.25) * TAU  # 6AM = horizon
    sun.rotation.x = sun_angle
    
    # Adjust light intensity (dim at night)
    var noon_factor := sin(time_of_day * PI)  # 0 at midnight, 1 at noon
    sun.light_energy = maxf(noon_factor, 0.0)
    
    # Adjust sky colors
    var env := world_env.environment
    if time_of_day < 0.25 or time_of_day > 0.75:
        # Night
        env.ambient_light_energy = 0.1
    else:
        # Day
        env.ambient_light_energy = 0.5
```

---

## Post-Processing

All post-processing is configured through the `Environment` resource:

```
Adjustments:
├── Brightness
├── Contrast
├── Saturation
└── Color correction LUT

Anti-Aliasing:
├── MSAA (geometry edges)
├── TAA (temporal, reduces shimmer)
└── FXAA (fast, slightly blurry)

Screen-Space Effects:
├── SSAO (ambient occlusion)
├── SSIL (indirect lighting)
├── SSR (reflections)
└── SS Contact Shadows

Glow/Bloom:
├── Intensity
├── Bloom amount
├── HDR threshold
└── Blend mode (additive, softlight, etc.)

Depth of Field:
├── Near blur (DOF)
└── Far blur (DOF)

Camera Effects:
├── DOF Bokeh shape
└── Motion blur (not yet in Godot 4 stable)
```

---

## 3D Optimization

### Draw Call Reduction

```
1. Use MultiMeshInstance3D for repeated objects (grass, trees, rocks)
2. Merge static meshes where possible
3. Use LOD (Level of Detail) — automatic in Godot 4.2+
4. Enable occlusion culling for indoor scenes
5. Reduce material count (share materials between meshes)
```

### MultiMesh for Instancing

```gdscript
# Draw 1000 trees with ONE draw call
var multi_mesh := MultiMesh.new()
multi_mesh.mesh = tree_mesh
multi_mesh.transform_format = MultiMesh.TRANSFORM_3D
multi_mesh.instance_count = 1000

for i in 1000:
    var transform := Transform3D()
    transform.origin = Vector3(
        randf_range(-100, 100),
        0,
        randf_range(-100, 100)
    )
    transform = transform.rotated(Vector3.UP, randf() * TAU)
    multi_mesh.set_instance_transform(i, transform)

var mmi := MultiMeshInstance3D.new()
mmi.multimesh = multi_mesh
add_child(mmi)
```

### Rendering Renderer Choice

```
Forward+ (default):
├── Full features, all effects
├── Good for: Desktop, console
└── Higher baseline cost

Mobile:
├── Simplified rendering
├── Good for: Mobile, low-end devices
└── Missing some advanced effects

Compatibility (GL):
├── OpenGL ES 3.0 / WebGL 2.0
├── Good for: Web exports, very old hardware
└── Most limited feature set
```

---

*← [07 — Audio System](./07-audio-system.md) | [09 — Navigation & AI](./09-navigation-and-ai.md) →*
