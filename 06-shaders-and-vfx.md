# 06 — Shaders & VFX

> Create stunning visual effects — from simple color changes to full screen post-processing.

---

## Table of Contents

- [Shader Basics](#shader-basics)
- [Shader Structure](#shader-structure)
- [2D Shader Recipes](#2d-shader-recipes)
- [3D Shader Recipes](#3d-shader-recipes)
- [Visual Shaders](#visual-shaders)
- [Post-Processing](#post-processing)
- [GPU Particles](#gpu-particles)
- [Performance Tips](#shader-performance)

---

## Shader Basics

Godot uses its own shader language based on GLSL (OpenGL ES 3.0). There are three shader types:

```
spatial   → 3D objects (meshes, materials)
canvas_item → 2D objects (sprites, UI, tilemaps)
particles → GPU particle systems
```

### Applying Shaders

1. Select a node (Sprite2D, MeshInstance3D, etc.)
2. In Inspector → Material → New ShaderMaterial
3. In the ShaderMaterial → Shader → New Shader
4. Click the shader to open the editor

### Shader vs ShaderMaterial

```
Shader         = The GPU program (code)
ShaderMaterial = Instance of a shader (with parameter values)

Multiple nodes can share the same Shader but each has its own
ShaderMaterial with different parameter values.
```

---

## Shader Structure

### 2D Canvas Item Shader

```glsl
shader_type canvas_item;

// Uniforms — parameters you can set from GDScript
uniform vec4 tint_color : source_color = vec4(1.0, 1.0, 1.0, 1.0);
uniform float intensity : hint_range(0.0, 1.0) = 0.5;
uniform sampler2D noise_texture;

// Vertex function — runs per vertex
void vertex() {
    // VERTEX is the position of this vertex
    // UV is the texture coordinate
    // TIME is the elapsed time in seconds
}

// Fragment function — runs per pixel
void fragment() {
    // COLOR is the output color
    // TEXTURE is the sprite's texture
    // UV is the texture coordinate (0,0 to 1,0)
    // SCREEN_UV is the screen coordinate
    // TIME is the elapsed time
    
    vec4 tex_color = texture(TEXTURE, UV);
    COLOR = mix(tex_color, tint_color, intensity);
}
```

### 3D Spatial Shader

```glsl
shader_type spatial;

uniform vec4 albedo_color : source_color = vec4(1.0);
uniform sampler2D albedo_texture : hint_default_white;
uniform float metallic : hint_range(0.0, 1.0) = 0.0;
uniform float roughness : hint_range(0.0, 1.0) = 0.5;

void fragment() {
    ALBEDO = texture(albedo_texture, UV).rgb * albedo_color.rgb;
    METALLIC = metallic;
    ROUGHNESS = roughness;
}
```

### Setting Uniforms from GDScript

```gdscript
# Get the material
var material := sprite.material as ShaderMaterial

# Set shader parameters
material.set_shader_parameter("tint_color", Color.RED)
material.set_shader_parameter("intensity", 0.8)
material.set_shader_parameter("noise_texture", noise_tex)

# Animate shader params with tweens
func dissolve() -> void:
    var mat := sprite.material as ShaderMaterial
    var tween := create_tween()
    tween.tween_method(
        func(val: float): mat.set_shader_parameter("dissolve_amount", val),
        0.0, 1.0, 1.0
    )
```

---

## 2D Shader Recipes

### Flash White (Hit Effect)

```glsl
shader_type canvas_item;

uniform float flash_amount : hint_range(0.0, 1.0) = 0.0;
uniform vec4 flash_color : source_color = vec4(1.0, 1.0, 1.0, 1.0);

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    COLOR = mix(tex, flash_color, flash_amount);
    COLOR.a = tex.a;  // Keep original transparency
}
```

### Outline

```glsl
shader_type canvas_item;

uniform vec4 outline_color : source_color = vec4(0.0, 0.0, 0.0, 1.0);
uniform float outline_width : hint_range(0.0, 10.0, 1.0) = 1.0;

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    vec2 size = TEXTURE_PIXEL_SIZE * outline_width;
    
    float outline = texture(TEXTURE, UV + vec2(-size.x, 0)).a;
    outline += texture(TEXTURE, UV + vec2(size.x, 0)).a;
    outline += texture(TEXTURE, UV + vec2(0, -size.y)).a;
    outline += texture(TEXTURE, UV + vec2(0, size.y)).a;
    // Diagonals for smoother outline
    outline += texture(TEXTURE, UV + vec2(-size.x, -size.y)).a;
    outline += texture(TEXTURE, UV + vec2(size.x, -size.y)).a;
    outline += texture(TEXTURE, UV + vec2(-size.x, size.y)).a;
    outline += texture(TEXTURE, UV + vec2(size.x, size.y)).a;
    outline = min(outline, 1.0);
    
    vec4 color = mix(outline_color * outline, tex, tex.a);
    COLOR = color;
}
```

### Dissolve Effect

```glsl
shader_type canvas_item;

uniform float dissolve_amount : hint_range(0.0, 1.0) = 0.0;
uniform sampler2D noise_texture;
uniform vec4 edge_color : source_color = vec4(1.0, 0.5, 0.0, 1.0);
uniform float edge_width : hint_range(0.0, 0.1) = 0.03;

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    float noise = texture(noise_texture, UV).r;
    
    float edge = smoothstep(dissolve_amount, dissolve_amount + edge_width, noise);
    float alpha = step(dissolve_amount, noise);
    
    vec4 final_color = mix(edge_color, tex, edge);
    final_color.a *= alpha * tex.a;
    COLOR = final_color;
}
```

### Chromatic Aberration

```glsl
shader_type canvas_item;

uniform float amount : hint_range(0.0, 0.01) = 0.003;

void fragment() {
    float r = texture(TEXTURE, UV + vec2(amount, 0.0)).r;
    float g = texture(TEXTURE, UV).g;
    float b = texture(TEXTURE, UV - vec2(amount, 0.0)).b;
    float a = texture(TEXTURE, UV).a;
    COLOR = vec4(r, g, b, a);
}
```

### Pixelation

```glsl
shader_type canvas_item;

uniform float pixel_size : hint_range(1.0, 64.0, 1.0) = 4.0;

void fragment() {
    vec2 size = vec2(textureSize(TEXTURE, 0));
    vec2 grid_uv = round(UV * size / pixel_size) * pixel_size / size;
    COLOR = texture(TEXTURE, grid_uv);
}
```

### Water / Wave Distortion

```glsl
shader_type canvas_item;

uniform float wave_speed : hint_range(0.0, 10.0) = 2.0;
uniform float wave_amplitude : hint_range(0.0, 0.1) = 0.02;
uniform float wave_frequency : hint_range(0.0, 50.0) = 10.0;

void fragment() {
    vec2 uv = UV;
    uv.x += sin(UV.y * wave_frequency + TIME * wave_speed) * wave_amplitude;
    uv.y += cos(UV.x * wave_frequency + TIME * wave_speed) * wave_amplitude * 0.5;
    COLOR = texture(TEXTURE, uv);
}
```

### Palette Swap

```glsl
shader_type canvas_item;

uniform sampler2D palette_old;  // 1-pixel-tall strip of original colors
uniform sampler2D palette_new;  // 1-pixel-tall strip of replacement colors
uniform float tolerance : hint_range(0.0, 0.1) = 0.02;

void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    vec2 palette_size = vec2(textureSize(palette_old, 0));
    
    for (float i = 0.0; i < palette_size.x; i += 1.0) {
        vec2 palette_uv = vec2((i + 0.5) / palette_size.x, 0.5);
        vec4 old_color = texture(palette_old, palette_uv);
        
        if (distance(tex.rgb, old_color.rgb) < tolerance) {
            vec4 new_color = texture(palette_new, palette_uv);
            COLOR = vec4(new_color.rgb, tex.a);
            return;
        }
    }
    COLOR = tex;
}
```

---

## 3D Shader Recipes

### Fresnel / Rim Light

```glsl
shader_type spatial;

uniform vec4 fresnel_color : source_color = vec4(0.0, 0.5, 1.0, 1.0);
uniform float fresnel_power : hint_range(0.1, 10.0) = 3.0;

void fragment() {
    float fresnel = pow(1.0 - dot(NORMAL, VIEW), fresnel_power);
    ALBEDO = vec3(0.1);
    EMISSION = fresnel_color.rgb * fresnel;
}
```

### Triplanar Mapping (for terrain)

```glsl
shader_type spatial;

uniform sampler2D texture_map : hint_default_white;
uniform float blend_sharpness : hint_range(1.0, 32.0) = 8.0;

void fragment() {
    vec3 world_normal = abs(normalize(NORMAL));
    vec3 blending = pow(world_normal, vec3(blend_sharpness));
    blending /= blending.x + blending.y + blending.z;
    
    vec3 x_projection = texture(texture_map, VERTEX.yz).rgb;
    vec3 y_projection = texture(texture_map, VERTEX.xz).rgb;
    vec3 z_projection = texture(texture_map, VERTEX.xy).rgb;
    
    ALBEDO = x_projection * blending.x + 
             y_projection * blending.y + 
             z_projection * blending.z;
}
```

---

## Visual Shaders

For those who prefer node-based shader creation:

1. Create ShaderMaterial → New **VisualShader**
2. Click to open the Visual Shader Editor
3. Right-click in the graph → Add Node
4. Connect nodes with wires

### Key Visual Shader Nodes

```
Input:
├── Input → UV              — Texture coordinates
├── Input → Time            — Elapsed time
├── Input → Screen UV       — Screen position
└── Texture → Texture2D     — Sample a texture

Math:
├── VectorOp → Mix          — Blend between values
├── VectorOp → Add/Multiply — Math operations
├── ScalarFunc → Sin/Cos    — Wave functions
└── ScalarOp → Step         — Threshold

Color:
├── Color → ColorConstant   — Solid color
└── Color → HSV             — Hue/Saturation/Value

Output:
├── Output → Color          — Final pixel color
├── Output → Alpha          — Transparency
└── Output → Emission       — Glow (3D)
```

---

## Post-Processing

### Full-Screen Shader Setup

```
Method 1: ColorRect covering the screen
1. Add a CanvasLayer (layer = highest)
2. Add ColorRect as child
3. Set anchors to PRESET_FULL_RECT
4. Add ShaderMaterial to ColorRect
5. Check "Show Behind Parent" on the CanvasLayer
```

### Vignette

```glsl
shader_type canvas_item;

uniform float vignette_intensity : hint_range(0.0, 1.0) = 0.4;
uniform float vignette_opacity : hint_range(0.0, 1.0) = 0.5;

void fragment() {
    // Get screen texture
    vec4 color = texture(TEXTURE, UV);
    
    // Vignette
    float vignette = UV.x * UV.y * (1.0 - UV.x) * (1.0 - UV.y);
    vignette = clamp(pow(16.0 * vignette, vignette_intensity), 0.0, 1.0);
    color.rgb = mix(color.rgb, color.rgb * vignette, vignette_opacity);
    
    COLOR = color;
}
```

### CRT / Retro Effect

```glsl
shader_type canvas_item;

uniform float scanline_strength : hint_range(0.0, 1.0) = 0.3;
uniform float curvature : hint_range(0.0, 0.1) = 0.03;
uniform float aberration : hint_range(0.0, 0.01) = 0.003;

void fragment() {
    // Screen curvature
    vec2 uv = UV * 2.0 - 1.0;
    uv *= 1.0 + pow(abs(uv.yx), vec2(2.0)) * curvature;
    uv = (uv + 1.0) * 0.5;
    
    // Chromatic aberration
    float r = texture(TEXTURE, uv + vec2(aberration, 0.0)).r;
    float g = texture(TEXTURE, uv).g;
    float b = texture(TEXTURE, uv - vec2(aberration, 0.0)).b;
    
    // Scanlines
    float scanline = sin(uv.y * 800.0) * 0.5 + 0.5;
    scanline = 1.0 - scanline_strength * scanline;
    
    // Border
    float border = step(0.0, uv.x) * step(uv.x, 1.0) * step(0.0, uv.y) * step(uv.y, 1.0);
    
    COLOR = vec4(vec3(r, g, b) * scanline * border, 1.0);
}
```

---

## GPU Particles

### GPUParticles2D Setup

```gdscript
# fire_particles.gd
extends GPUParticles2D

func _ready() -> void:
    emitting = true
    amount = 50
    lifetime = 1.0
    
    # Use a ParticleProcessMaterial
    var mat := ParticleProcessMaterial.new()
    mat.direction = Vector3(0, -1, 0)  # Up
    mat.initial_velocity_min = 50.0
    mat.initial_velocity_max = 100.0
    mat.gravity = Vector3(0, -50, 0)
    mat.scale_min = 0.5
    mat.scale_max = 1.0
    
    # Color gradient
    var gradient := Gradient.new()
    gradient.set_color(0, Color(1.0, 0.8, 0.0))
    gradient.set_color(1, Color(1.0, 0.0, 0.0, 0.0))
    var grad_tex := GradientTexture1D.new()
    grad_tex.gradient = gradient
    mat.color_ramp = grad_tex
    
    process_material = mat
```

### One-Shot Particle Burst

```gdscript
func explosion_at(pos: Vector2) -> void:
    var particles := GPUParticles2D.new()
    particles.global_position = pos
    particles.emitting = true
    particles.one_shot = true
    particles.amount = 100
    particles.lifetime = 0.5
    particles.explosiveness = 1.0  # All at once
    
    # Auto-cleanup after particles finish
    get_tree().root.add_child(particles)
    await get_tree().create_timer(particles.lifetime + 0.5).timeout
    particles.queue_free()
```

---

## Shader Performance

```
DO:
✅ Use simpler shaders when possible
✅ Cache texture lookups
✅ Use step() instead of if/else
✅ Prefer built-in functions (faster than manual math)
✅ Use hint_* qualifiers on uniforms

DON'T:
❌ Sample textures in loops
❌ Use discard excessively (breaks optimizations)
❌ Use complex conditionals (GPUs hate branching)
❌ Use high-res noise textures when low-res works
```

---

*← [05 — UI & Themes](./05-ui-and-themes.md) | [07 — Audio System](./07-audio-system.md) →*
