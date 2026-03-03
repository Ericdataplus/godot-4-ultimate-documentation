# 04 — Animation System

> From simple sprite flips to complex 3D blend trees — master every animation tool Godot offers.

---

## Table of Contents

- [AnimationPlayer Basics](#animationplayer-basics)
- [Keyframing Everything](#keyframing-everything)
- [Animation Callbacks & Events](#animation-callbacks--events)
- [AnimatedSprite2D](#animatedsprite2d)
- [AnimationTree](#animationtree)
- [State Machines in AnimationTree](#state-machines)
- [Blend Trees](#blend-trees)
- [BlendSpace1D & BlendSpace2D](#blendspaces)
- [Tweens — Procedural Animation](#tweens)
- [Programmatic Animation Recipes](#recipes)

---

## AnimationPlayer Basics

AnimationPlayer can animate **ANY** property on **ANY** node. Position, color, shader params, even function calls.

### Creating Animations

1. Add an `AnimationPlayer` node
2. Click the Animation panel at bottom
3. Click "Animation" → "New" → name it
4. Select a node, find a property in Inspector
5. Click the key icon 🔑 next to a property to add a keyframe
6. Move the timeline cursor, change the property, keyframe again

### Controlling from Code

```gdscript
@onready var anim: AnimationPlayer = $AnimationPlayer

func _ready() -> void:
    # Play animation
    anim.play("idle")
    
    # Play from specific time
    anim.play("walk")
    anim.seek(0.5)  # Jump to 0.5 seconds
    
    # Play backwards
    anim.play_backwards("attack")
    # Or: anim.play("attack", -1, -1.0)
    
    # Speed control
    anim.play("run")
    anim.speed_scale = 2.0  # Double speed
    
    # Queue animations
    anim.play("attack")
    anim.queue("idle")  # Plays after attack finishes
    
    # Check current animation
    if anim.current_animation == "attack":
        pass
    
    # Check if playing
    if anim.is_playing():
        pass

# Wait for animation to finish
func play_attack() -> void:
    anim.play("attack")
    await anim.animation_finished
    anim.play("idle")

# Connect to signals
func _ready() -> void:
    anim.animation_finished.connect(_on_animation_finished)
    anim.animation_started.connect(_on_animation_started)

func _on_animation_finished(anim_name: StringName) -> void:
    match anim_name:
        "attack":
            anim.play("idle")
        "death":
            queue_free()
```

### Animation Library

Godot 4 uses Animation Libraries to organize animations:

```gdscript
# Default library (no prefix)
anim.play("idle")

# Named library
anim.play("combat/slash")    # Library "combat", animation "slash"
anim.play("movement/run")    # Library "movement", animation "run"
```

---

## Keyframing Everything

Literally any property can be animated:

```
Sprite2D:
├── position           → Move
├── rotation           → Rotate
├── scale              → Scale (squash & stretch!)
├── modulate           → Color/transparency
├── frame              → Sprite sheet frame
├── visible            → Show/hide
└── material:shader_parameter/dissolve → Shader values!

Control nodes:
├── position           → UI animation
├── size               → Resize
├── modulate:a         → Fade in/out
├── custom_minimum_size → Grow/shrink
└── theme_override_colors/font_color → Color change

AudioStreamPlayer:
├── volume_db          → Volume fade
└── pitch_scale        → Pitch shift

Light2D:
├── energy             → Brightness
├── color              → Color
└── texture_scale      → Size
```

### Easing & Interpolation

Right-click on a keyframe in the animation editor to set its easing:

```
Linear          — Constant speed (default)
Ease In         — Starts slow, speeds up
Ease Out        — Starts fast, slows down
Ease In-Out     — Slow start and end
Cubic           — Smooth curves
Bounce          — Bouncing effect
Elastic         — Spring/rubber band
```

---

## Animation Callbacks & Events

### Method Call Track

Add a "Call Method" track to trigger functions at specific times:

```gdscript
# These methods are called by the AnimationPlayer at keyframed times
func spawn_particles() -> void:
    $Particles.emitting = true

func play_sound() -> void:
    $AudioStreamPlayer.play()

func deal_damage() -> void:
    var enemies := $HitBox.get_overlapping_bodies()
    for enemy in enemies:
        if enemy.has_method("take_damage"):
            enemy.take_damage(10)

func enable_hitbox() -> void:
    $HitBox/CollisionShape2D.set_deferred("disabled", false)

func disable_hitbox() -> void:
    $HitBox/CollisionShape2D.set_deferred("disabled", true)
```

### Typical Attack Animation Timeline

```
Time: 0.0    0.1    0.2    0.3    0.4    0.5    0.6
      |------|------|------|------|------|------|
      Start  Wind   Enable  Hit   Disable  End
      Anim   Up     Hitbox  Frame Hitbox   Idle

Method calls at:
  0.2 → enable_hitbox()
  0.3 → deal_damage(), spawn_particles(), play_sound()
  0.4 → disable_hitbox()
```

---

## AnimatedSprite2D

For sprite sheet animations — simpler than AnimationPlayer for basic cases:

```gdscript
@onready var sprite: AnimatedSprite2D = $AnimatedSprite2D

func _physics_process(delta: float) -> void:
    if velocity.length() > 10:
        sprite.play("walk")
    else:
        sprite.play("idle")
    
    # Flip based on direction
    if velocity.x != 0:
        sprite.flip_h = velocity.x < 0

# Signals
func _ready() -> void:
    sprite.animation_finished.connect(_on_animation_finished)
    sprite.frame_changed.connect(_on_frame_changed)

func _on_animation_finished() -> void:
    if sprite.animation == "attack":
        sprite.play("idle")

func _on_frame_changed() -> void:
    # Do something on specific frames
    if sprite.animation == "attack" and sprite.frame == 3:
        deal_damage()
```

### SpriteFrames Tips

```gdscript
# Programmatic sprite frame control
var frames := sprite.sprite_frames
frames.get_frame_count("idle")        # Number of frames
frames.get_animation_speed("walk")    # FPS for this animation
frames.has_animation("jump")          # Check if animation exists
```

---

## AnimationTree

For complex animation blending and state management. **Required for smooth character animation.**

### Setup

1. Add `AnimationTree` node as sibling (not child) of `AnimationPlayer`
2. Set `Anim Player` property to your AnimationPlayer
3. Set `Active` to `true`
4. Choose a root node type:
   - **AnimationNodeStateMachine** — for state-based animation
   - **AnimationNodeBlendTree** — for blending animations
   - **AnimationNodeBlendSpace2D** — for directional blending

### Important: Disable AnimationPlayer Direct Playback

When using AnimationTree, **don't call `anim_player.play()` directly** — it conflicts with the tree. Only control the AnimationTree.

---

## State Machines

The most common AnimationTree setup:

```
AnimationTree (root: AnimationNodeStateMachine)
├── Idle → (auto-advance to) → Walk → Run
├── Jump
├── Fall
├── Attack
└── Death
```

### Controlling the State Machine

```gdscript
@onready var anim_tree: AnimationTree = $AnimationTree
@onready var state_machine: AnimationNodeStateMachinePlayback

func _ready() -> void:
    state_machine = anim_tree["parameters/playback"] as AnimationNodeStateMachinePlayback

func _physics_process(delta: float) -> void:
    # Travel to a state (with transition)
    if is_on_floor():
        if velocity.length() > 10:
            state_machine.travel("walk")
        else:
            state_machine.travel("idle")
    else:
        if velocity.y < 0:
            state_machine.travel("jump")
        else:
            state_machine.travel("fall")

func attack() -> void:
    # Force immediate state change (no transition)
    state_machine.travel("attack")
    # OR start immediately:
    state_machine.start("attack")
    
    await anim_tree.animation_finished
    state_machine.travel("idle")

# Check current state
func get_current_state() -> String:
    return state_machine.get_current_node()
```

### Transition Types

```
Immediate   — Cut instantly to new animation
Xfade       — Cross-fade over time (smooth blend)
Sync        — Match position in timeline (for looping anims)

Switch Modes:
At End      — Wait for current animation to finish before transitioning
              (Great for attacks — ensures full animation plays)
Immediate   — Transition happens right away
Auto        — Transition automatically when conditions are met

Configure in the AnimationTree editor by clicking transition arrows
```

### AnimationTree Tips (Community-Sourced)

```gdscript
# 1. USE OneShot NODES for interruptible actions (hit reactions, attacks)
# AnimationNodeOneShot plays an animation once then returns to the mix
anim_tree["parameters/OneShot/request"] = AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE

# 2. DRAG parameters from AnimationTree panel into script
# Get the exact path for any parameter without guessing

# 3. SAVE BlendSpace configs as resources
# Right-click BlendSpace in AnimationTree → Save
# Reuse across different characters with the same animation set

# 4. SPEED SCALING from code
anim_tree["parameters/TimeScale/scale"] = velocity.length() / max_speed
# Walk animation speed matches character movement speed

# 5. DON'T call AnimationPlayer.play() when AnimationTree is active!
# It conflicts. Only control through AnimationTree parameters.

# 6. ACTIVATION in code
func _ready() -> void:
    anim_tree.active = true  # Must be active to work!
```

---

## Blend Trees

Mix multiple animations smoothly:

```gdscript
# Blend between idle and walk based on speed
@onready var anim_tree: AnimationTree = $AnimationTree

func _physics_process(delta: float) -> void:
    var speed_ratio := velocity.length() / max_speed
    anim_tree["parameters/IdleWalkBlend/blend_amount"] = speed_ratio
```

---

## BlendSpaces

### BlendSpace1D — One Axis

Perfect for speed-based blending (idle → walk → run):

```gdscript
# Blend between idle(0), walk(0.5), run(1) based on speed
func _physics_process(delta: float) -> void:
    var blend := clampf(velocity.length() / max_speed, 0.0, 1.0)
    anim_tree["parameters/BlendSpace1D/blend_position"] = blend
```

### BlendSpace2D — Two Axes

Perfect for directional movement (8-directional animation):

```gdscript
# Blend between directional walk animations
func _physics_process(delta: float) -> void:
    if velocity.length() > 10:
        var blend_pos := velocity.normalized()
        anim_tree["parameters/BlendSpace2D/blend_position"] = blend_pos
```

Setup in the editor:
```
        walk_up (0, -1)
           ▲
           │
walk_left ◄─┼─► walk_right
(-1, 0)    │    (1, 0)
           ▼
      walk_down (0, 1)
```

Add idle at (0, 0) for a 5-point blend that handles standing still.

### BlendSpace Tips

```gdscript
# For 2D PIXEL ART: Use DISCRETE blend mode
# Discrete = snap to nearest animation (no blending between sprites)
# Continuous = smooth blend (better for 3D skeletal animation)
# Set in BlendSpace2D → Blend Mode → Discrete

# For 3D: Use CONTINUOUS (default) for smooth skeletal blending

# Save BlendSpaces as reusable resources:
# Right-click → Save → .tres file
# Load into different characters' AnimationTrees
```

---

## Tweens

Tweens are **procedural animations** — create them in code, no editor needed.

### Basic Tween Usage

```gdscript
# Simple property tween
func slide_in() -> void:
    var tween := create_tween()
    tween.tween_property(self, "position:x", 500.0, 0.5)

# Multiple properties in sequence
func death_animation() -> void:
    var tween := create_tween()
    tween.tween_property(sprite, "modulate", Color.RED, 0.1)
    tween.tween_property(sprite, "modulate:a", 0.0, 0.3)
    tween.tween_callback(queue_free)

# Parallel tweens (happen at the same time)
func entrance() -> void:
    var tween := create_tween()
    tween.set_parallel(true)
    tween.tween_property(self, "position:y", 300.0, 0.5)
    tween.tween_property(self, "modulate:a", 1.0, 0.5)
    tween.tween_property(self, "scale", Vector2.ONE, 0.3)

# Chain parallel and sequential
func complex_animation() -> void:
    var tween := create_tween()
    
    # Step 1: Slide in (parallel)
    tween.set_parallel(true)
    tween.tween_property(self, "position:x", 200, 0.3)
    tween.tween_property(self, "modulate:a", 1.0, 0.3)
    
    # Step 2: Then bounce (sequential again)
    tween.set_parallel(false)
    tween.tween_property(self, "scale", Vector2(1.2, 0.8), 0.1)
    tween.tween_property(self, "scale", Vector2(0.9, 1.1), 0.1)
    tween.tween_property(self, "scale", Vector2.ONE, 0.1)
```

### Tween Easing

```gdscript
func smooth_move() -> void:
    var tween := create_tween()
    tween.tween_property(self, "position", target, 0.5)\
        .set_ease(Tween.EASE_OUT)\
        .set_trans(Tween.TRANS_CUBIC)

# Common combinations:
# Slide in:    EASE_OUT + TRANS_CUBIC (decelerates)
# Slide out:   EASE_IN + TRANS_CUBIC (accelerates)
# Bounce:      EASE_OUT + TRANS_BOUNCE
# Elastic:     EASE_OUT + TRANS_ELASTIC (spring)
# Smooth:      EASE_IN_OUT + TRANS_SINE
# Snappy:      EASE_OUT + TRANS_EXPO (fast then slow)
```

### Tween Patterns

```gdscript
# Looping tween (hover effect)
func hover_float() -> void:
    var tween := create_tween().set_loops()
    tween.tween_property(self, "position:y", position.y - 10, 0.8)\
        .set_ease(Tween.EASE_IN_OUT).set_trans(Tween.TRANS_SINE)
    tween.tween_property(self, "position:y", position.y + 10, 0.8)\
        .set_ease(Tween.EASE_IN_OUT).set_trans(Tween.TRANS_SINE)

# Interval (delay between steps)
func typewriter_text(label: Label, text: String) -> void:
    label.text = ""
    var tween := create_tween()
    for i in text.length():
        tween.tween_callback(func(): label.text += text[i])
        tween.tween_interval(0.03)

# Kill previous tween before starting new one
var current_tween: Tween
func flash() -> void:
    if current_tween:
        current_tween.kill()
    current_tween = create_tween()
    current_tween.tween_property(sprite, "modulate", Color.WHITE, 0.2)\
        .from(Color.RED)

# Tween with callback
func collect_coin() -> void:
    var tween := create_tween()
    tween.tween_property(self, "position:y", position.y - 50, 0.3)
    tween.parallel().tween_property(self, "modulate:a", 0.0, 0.3)
    tween.tween_callback(queue_free)  # Delete when done
```

---

## Recipes

### Screen Shake

```gdscript
# camera_shake.gd — attach to Camera2D
extends Camera2D

var shake_strength: float = 0.0
var shake_decay: float = 5.0

func shake(intensity: float = 10.0, duration: float = 0.3) -> void:
    shake_strength = intensity
    var tween := create_tween()
    tween.tween_property(self, "shake_strength", 0.0, duration)

func _process(delta: float) -> void:
    if shake_strength > 0.1:
        offset = Vector2(
            randf_range(-shake_strength, shake_strength),
            randf_range(-shake_strength, shake_strength)
        )
    else:
        offset = Vector2.ZERO
```

### Squash & Stretch

```gdscript
func squash_stretch() -> void:
    var tween := create_tween()
    tween.tween_property(sprite, "scale", Vector2(1.3, 0.7), 0.05)
    tween.tween_property(sprite, "scale", Vector2(0.8, 1.2), 0.1)
    tween.tween_property(sprite, "scale", Vector2.ONE, 0.15)\
        .set_ease(Tween.EASE_OUT).set_trans(Tween.TRANS_ELASTIC)
```

### Hit Stop (Freeze Frame)

```gdscript
func hit_stop(duration: float = 0.05) -> void:
    Engine.time_scale = 0.05
    await get_tree().create_timer(duration * 0.05).timeout
    Engine.time_scale = 1.0
```

### Damage Number Popup

```gdscript
# damage_number.gd
extends Label

func _ready() -> void:
    text = ""

func show_damage(amount: int, crit: bool = false) -> void:
    text = str(amount)
    modulate = Color.YELLOW if crit else Color.WHITE
    scale = Vector2(1.5, 1.5) if crit else Vector2.ONE
    
    var tween := create_tween()
    tween.set_parallel(true)
    tween.tween_property(self, "position:y", position.y - 60, 0.5)\
        .set_ease(Tween.EASE_OUT)
    tween.tween_property(self, "modulate:a", 0.0, 0.5)\
        .set_ease(Tween.EASE_IN).set_delay(0.3)
    tween.tween_property(self, "scale", Vector2.ONE * 0.5, 0.5)
    
    tween.set_parallel(false)
    tween.tween_callback(queue_free)
```

---

*← [03 — Physics & Collision](./03-physics-and-collision.md) | [05 — UI & Themes](./05-ui-and-themes.md) →*
