# 07 — Audio System

> Music, sound effects, audio buses, and positional audio done right.

---

## Table of Contents

- [Audio Nodes](#audio-nodes)
- [Audio Buses](#audio-buses)
- [AudioManager Pattern](#audiomanager-pattern)
- [Positional Audio](#positional-audio)
- [Audio Bus Effects](#audio-bus-effects)
- [Music Crossfading](#music-crossfading)
- [Sound Variation](#sound-variation)
- [Audio Formats](#audio-formats)

---

## Audio Nodes

```
AudioStreamPlayer    — Non-positional (music, UI sounds, ambient)
AudioStreamPlayer2D  — Positional 2D (footsteps, enemy sounds)
AudioStreamPlayer3D  — Positional 3D (environmental sounds)
```

### Basic Usage

```gdscript
# Play a sound
@onready var sfx: AudioStreamPlayer = $AudioStreamPlayer

func _ready() -> void:
    sfx.stream = preload("res://audio/sfx/jump.wav")

func play_jump_sound() -> void:
    sfx.play()

# With volume control
sfx.volume_db = -6.0  # 0 = max, negative = quieter

# With pitch variation (for variety)
sfx.pitch_scale = randf_range(0.9, 1.1)

# Check if playing
if not sfx.playing:
    sfx.play()

# Wait for sound to finish
sfx.play()
await sfx.finished
print("Sound finished!")
```

---

## Audio Buses

Audio buses group sounds for independent volume control. **Set this up early!**

### Recommended Bus Layout

```
Master (always exists)
├── Music        — Background music, ambient tracks
├── SFX          — Sound effects, impacts, UI clicks  
│   ├── Voice    — Dialog, voice lines (sub-bus of SFX)
│   └── Environment — Wind, rain, ambient loops
└── UI           — Menu sounds, button clicks
```

### Creating Buses

1. Open the **Audio** tab at the bottom of the editor
2. Click **Add Bus** for each category
3. Name them: Music, SFX, UI
4. Route sub-buses by setting their **Send** property

### Assigning Sounds to Buses

```gdscript
# In the Inspector: Bus property → select "Music", "SFX", etc.
# Or in code:
audio_player.bus = "SFX"
audio_player.bus = "Music"
```

### Volume Control (Settings Menu)

```gdscript
# Set bus volume (linear 0.0 to 1.0 → converted to dB)
func set_bus_volume(bus_name: String, linear_value: float) -> void:
    var bus_index := AudioServer.get_bus_index(bus_name)
    AudioServer.set_bus_volume_db(bus_index, linear_to_db(linear_value))

# Mute/unmute a bus
func toggle_mute(bus_name: String) -> void:
    var bus_index := AudioServer.get_bus_index(bus_name)
    var muted := AudioServer.is_bus_mute(bus_index)
    AudioServer.set_bus_mute(bus_index, !muted)

# Get current volume (for slider initialization)
func get_bus_volume(bus_name: String) -> float:
    var bus_index := AudioServer.get_bus_index(bus_name)
    return db_to_linear(AudioServer.get_bus_volume_db(bus_index))
```

### Saving Bus Layout

> **⚠️ Important:** Save your bus layout! Go to Audio tab → Click the three dots → Save As → `default_bus_layout.tres`
> Set it in Project Settings → Audio → Buses → Default Bus Layout

---

## AudioManager Pattern

A global autoload that manages all audio playback:

```gdscript
# audio_manager.gd (set as Autoload → "AudioManager")
extends Node

# Music players (two for crossfading)
var music_a: AudioStreamPlayer
var music_b: AudioStreamPlayer
var active_music: AudioStreamPlayer

# Pool of SFX players
var sfx_pool: Array[AudioStreamPlayer] = []
const SFX_POOL_SIZE: int = 16

# Preloaded sounds
var sounds: Dictionary = {}

func _ready() -> void:
    # Create music players
    music_a = AudioStreamPlayer.new()
    music_a.bus = "Music"
    add_child(music_a)
    
    music_b = AudioStreamPlayer.new()
    music_b.bus = "Music"
    add_child(music_b)
    
    active_music = music_a
    
    # Create SFX pool
    for i in SFX_POOL_SIZE:
        var player := AudioStreamPlayer.new()
        player.bus = "SFX"
        add_child(player)
        sfx_pool.append(player)

# ─── SFX ───

func play_sfx(stream: AudioStream, volume_db: float = 0.0, pitch: float = 1.0) -> void:
    var player := _get_available_sfx_player()
    if player == null:
        return
    player.stream = stream
    player.volume_db = volume_db
    player.pitch_scale = pitch
    player.play()

func play_sfx_random_pitch(stream: AudioStream, pitch_range: float = 0.1) -> void:
    play_sfx(stream, 0.0, randf_range(1.0 - pitch_range, 1.0 + pitch_range))

func _get_available_sfx_player() -> AudioStreamPlayer:
    for player in sfx_pool:
        if not player.playing:
            return player
    return null  # All busy

# ─── MUSIC ───

func play_music(stream: AudioStream, fade_duration: float = 1.0) -> void:
    if active_music.stream == stream and active_music.playing:
        return  # Already playing this track
    
    var new_player := music_b if active_music == music_a else music_a
    new_player.stream = stream
    new_player.volume_db = -80.0
    new_player.play()
    
    # Crossfade
    var tween := create_tween().set_parallel(true)
    tween.tween_property(active_music, "volume_db", -80.0, fade_duration)
    tween.tween_property(new_player, "volume_db", 0.0, fade_duration)
    
    tween.set_parallel(false)
    tween.tween_callback(func():
        active_music.stop()
        active_music = new_player
    )

func stop_music(fade_duration: float = 1.0) -> void:
    var tween := create_tween()
    tween.tween_property(active_music, "volume_db", -80.0, fade_duration)
    tween.tween_callback(active_music.stop)
```

Usage from anywhere:
```gdscript
# Play SFX
var jump_sfx := preload("res://audio/sfx/jump.wav")
AudioManager.play_sfx(jump_sfx)
AudioManager.play_sfx_random_pitch(jump_sfx)

# Play music
var battle_music := preload("res://audio/music/battle.ogg")
AudioManager.play_music(battle_music, 2.0)  # 2-second crossfade
```

---

## Positional Audio

### 2D Positional Audio

```gdscript
# AudioStreamPlayer2D automatically adjusts volume and panning
# based on distance from the AudioListener2D (usually the Camera2D)

# Key properties:
audio_2d.max_distance = 2000.0  # Beyond this = silent
audio_2d.attenuation = 1.0      # 1.0 = linear falloff, higher = faster falloff
audio_2d.max_polyphony = 4      # Max overlapping sounds
```

### 3D Positional Audio

```gdscript
# AudioStreamPlayer3D uses 3D distance attenuation
audio_3d.unit_size = 10.0       # Distance for full volume
audio_3d.max_distance = 50.0    # Beyond this = silent
audio_3d.max_db = 3.0           # Volume boost at close range
audio_3d.attenuation_model = AudioStreamPlayer3D.ATTENUATION_INVERSE_DISTANCE
```

---

## Audio Bus Effects

Add effects to buses for atmosphere:

```
Reverb       — Echoing rooms, caves, cathedrals
Delay        — Echo repeat effects
Chorus       — Thickens sound (subtle doubling)
EQ           — Adjust frequency bands
LowPassFilter — Muffle sounds (underwater, through walls)
HighPassFilter — Tinny, radio-like sound
Compressor   — Even out volume differences
Limiter      — Prevent clipping
Distortion   — Gritty, overdriven sound
Phaser       — Sweeping phase effect
```

### Dynamic Bus Effects (Gameplay)

```gdscript
# Underwater effect — add LowPass to Master bus
func enter_water() -> void:
    var effect := AudioEffectLowPassFilter.new()
    effect.cutoff_hz = 800.0
    AudioServer.add_bus_effect(0, effect)  # 0 = Master bus

func exit_water() -> void:
    # Remove the last added effect
    var effect_count := AudioServer.get_bus_effect_count(0)
    if effect_count > 0:
        AudioServer.remove_bus_effect(0, effect_count - 1)

# Pause menu — lower all game audio
func pause_audio() -> void:
    var sfx_idx := AudioServer.get_bus_index("SFX")
    AudioServer.set_bus_volume_db(sfx_idx, -20.0)

# Slow-motion — lower pitch
func slow_motion() -> void:
    AudioServer.playback_speed_scale = 0.5
```

---

## Music Crossfading

```gdscript
# Transition between combat and explore music
enum MusicState { EXPLORE, COMBAT, BOSS }

func set_music_state(state: MusicState) -> void:
    match state:
        MusicState.EXPLORE:
            AudioManager.play_music(explore_music, 2.0)
        MusicState.COMBAT:
            AudioManager.play_music(combat_music, 0.5)
        MusicState.BOSS:
            AudioManager.play_music(boss_music, 1.0)
```

---

## Sound Variation

### AudioStreamRandomizer

Godot 4 has a built-in resource for random sound variation:

1. In Inspector → Stream → New AudioStreamRandomizer
2. Add multiple audio streams
3. Set random pitch and volume range
4. Plays a random sample each time

### Manual Variation

```gdscript
var footstep_sounds: Array[AudioStream] = [
    preload("res://audio/sfx/footstep_1.wav"),
    preload("res://audio/sfx/footstep_2.wav"),
    preload("res://audio/sfx/footstep_3.wav"),
    preload("res://audio/sfx/footstep_4.wav"),
]

var last_footstep: int = -1

func play_footstep() -> void:
    # Avoid repeating the same sound
    var idx: int = last_footstep
    while idx == last_footstep:
        idx = randi() % footstep_sounds.size()
    last_footstep = idx
    
    AudioManager.play_sfx_random_pitch(footstep_sounds[idx], 0.15)
```

---

## Audio Formats

| Format | Use Case | Loop | Size | Quality |
|--------|----------|------|------|---------|
| `.wav` | SFX, short sounds | ❌ by default | Large | Lossless |
| `.ogg` | Music, long audio | ✅ easy | Small | Good |
| `.mp3` | Music (alt) | ✅ | Small | Good |

### Import Settings

- **SFX (.wav):** Loop = Off, Force mono for pool efficiency
- **Music (.ogg):** Loop = On (set in Import dock), keep stereo
- **Ambient (.ogg):** Loop = On, can be mono

> **💡 Tip:** For music loops, set the loop point in the Import dock → Loop Mode = Loop. Make sure your audio file seamlessly loops!

---

*← [06 — Shaders & VFX](./06-shaders-and-vfx.md) | [08 — 3D Rendering](./08-3d-rendering.md) →*
