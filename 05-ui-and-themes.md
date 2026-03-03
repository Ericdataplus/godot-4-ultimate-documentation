# 05 — UI & Themes

> Build production-quality user interfaces with Godot's powerful Control system.

---

## Table of Contents

- [Control Node Basics](#control-node-basics)
- [Anchors & Margins](#anchors--margins)
- [Container Nodes](#container-nodes)
- [Common UI Patterns](#common-ui-patterns)
- [Theme System](#theme-system)
- [Theme Type Variations](#theme-type-variations)
- [Responsive UI](#responsive-ui)
- [UI Animation](#ui-animation)
- [UI Focus & Navigation](#ui-focus--navigation)
- [Practical Examples](#practical-examples)

---

## Control Node Basics

All UI nodes inherit from `Control`. Unlike Node2D, Control nodes use **anchors**, **margins**, and **containers** for layout.

### Essential Control Nodes

```
Control (base)
├── Label                — Text display
├── RichTextLabel        — Formatted text (BBCode, links, images)
├── Button               — Clickable button
├── TextureButton        — Image-based button
├── TextureRect          — Display textures/images
├── TextEdit             — Multi-line text input
├── LineEdit             — Single-line text input
├── ProgressBar          — Health bars, loading bars
├── HSlider / VSlider    — Slider controls
├── SpinBox              — Number input with arrows
├── CheckBox / CheckButton
├── OptionButton         — Dropdown select
├── Panel                — Background panel
├── ColorRect            — Solid color rectangle
├── NinePatchRect        — Scalable bordered images
├── TabContainer         — Tabbed views
├── ScrollContainer      — Scrollable content
└── MarginContainer      — Add padding
```

### Container Nodes (Auto-Layout)

```
HBoxContainer    — Horizontal layout (like CSS flexbox row)
VBoxContainer    — Vertical layout (like CSS flexbox column)
GridContainer    — Grid layout (like CSS grid)
FlowContainer   — Wrapping flow layout
CenterContainer — Centers single child
MarginContainer — Adds margins/padding around child
PanelContainer  — Panel with auto-sizing to child
HSplitContainer — Resizable horizontal split
VSplitContainer — Resizable vertical split
TabContainer    — Tabbed content
```

---

## Anchors & Margins

Anchors define where a control is "attached" relative to its parent:

```
Anchors range from 0.0 to 1.0

(0,0)────────────────(1,0)
  │                    │
  │    Anchor Points   │
  │                    │
  │    (0.5, 0.5)      │
  │      Center        │
  │                    │
(0,1)────────────────(1,1)
```

### Preset Anchors

```gdscript
# Common presets (use these!)
control.set_anchors_preset(Control.PRESET_TOP_LEFT)       # Default
control.set_anchors_preset(Control.PRESET_CENTER)          # Centered
control.set_anchors_preset(Control.PRESET_FULL_RECT)       # Fill parent
control.set_anchors_preset(Control.PRESET_CENTER_BOTTOM)   # Centered bottom
control.set_anchors_preset(Control.PRESET_TOP_WIDE)        # Full width, top

# With margin reset
control.set_anchors_preset(Control.PRESET_FULL_RECT, true)  # true = keep margins
```

### Size Flags (Inside Containers)

```gdscript
# Tell containers how this node should behave
control.size_flags_horizontal = Control.SIZE_FILL       # Fill available space
control.size_flags_horizontal = Control.SIZE_EXPAND     # Expand to fill
control.size_flags_horizontal = Control.SIZE_EXPAND_FILL # Both (most common)
control.size_flags_horizontal = Control.SIZE_SHRINK_BEGIN # Align start
control.size_flags_horizontal = Control.SIZE_SHRINK_CENTER # Center
control.size_flags_horizontal = Control.SIZE_SHRINK_END  # Align end

# Stretch ratio (relative size within container)
control.size_flags_stretch_ratio = 2.0  # Take 2x the space of ratio=1.0
```

---

## Container Nodes

### VBoxContainer — The Vertical Stack

```
VBoxContainer
├── Label ("Settings")         ← takes minimum height
├── HSlider (Volume)           ← takes minimum height  
├── CheckBox (Fullscreen)      ← takes minimum height
├── Button (Apply)             ← takes minimum height
└── [empty space]              ← remaining space
```

### GridContainer — The Grid

```gdscript
# Set columns in Inspector or code
grid.columns = 3

# Children fill left-to-right, top-to-bottom:
# [Item1] [Item2] [Item3]
# [Item4] [Item5] [Item6]
# [Item7] [Item8] [Item9]
```

### MarginContainer — Add Padding

```gdscript
# Set margins in Inspector under "Theme Override → Constants"
# margin_left, margin_right, margin_top, margin_bottom
# Or create via theme
```

### Practical Layout: Game HUD

```
CanvasLayer (layer 1 — always on top)
└── Control (Full Rect anchor)
    ├── MarginContainer (Full Rect, margins: 16px)
    │   ├── VBoxContainer
    │   │   ├── HBoxContainer (top bar)
    │   │   │   ├── HealthBar (EXPAND_FILL)
    │   │   │   ├── ManaBar (EXPAND_FILL)
    │   │   │   └── Label "Score: 0" (SHRINK_END)
    │   │   ├── Control (spacer — EXPAND_FILL, takes remaining space)
    │   │   └── HBoxContainer (bottom bar)
    │   │       ├── TextureRect (minimap)
    │   │       └── VBoxContainer (inventory hotbar)
    │   │           └── HBoxContainer
    │   │               ├── TextureRect (slot 1)
    │   │               ├── TextureRect (slot 2)
    │   │               └── TextureRect (slot 3)
```

---

## Common UI Patterns

### Health Bar

```gdscript
# health_bar.gd
extends ProgressBar

@export var health_component: HealthComponent

func _ready() -> void:
    if health_component:
        health_component.health_changed.connect(_on_health_changed)
        max_value = health_component.max_health
        value = health_component.health

func _on_health_changed(current: int, maximum: int) -> void:
    max_value = maximum
    # Smooth transition
    var tween := create_tween()
    tween.tween_property(self, "value", float(current), 0.3)\
        .set_ease(Tween.EASE_OUT).set_trans(Tween.TRANS_CUBIC)
```

### Dialog Box

```gdscript
# dialog_box.gd
extends PanelContainer

signal dialog_finished

@onready var name_label: Label = $VBox/NameLabel
@onready var text_label: RichTextLabel = $VBox/TextLabel
@onready var continue_indicator: Label = $VBox/ContinueIndicator

var dialog_lines: Array[Dictionary] = []
var current_line: int = 0
var is_typing: bool = false
var typing_speed: float = 0.03

func show_dialog(lines: Array[Dictionary]) -> void:
    dialog_lines = lines
    current_line = 0
    visible = true
    _display_line()

func _display_line() -> void:
    if current_line >= dialog_lines.size():
        _close()
        return
    
    var line := dialog_lines[current_line]
    name_label.text = line.get("speaker", "")
    text_label.text = line.get("text", "")
    text_label.visible_characters = 0
    continue_indicator.visible = false
    is_typing = true
    
    # Typewriter effect
    var tween := create_tween()
    tween.tween_property(
        text_label, "visible_characters",
        text_label.text.length(), 
        text_label.text.length() * typing_speed
    )
    tween.tween_callback(func():
        is_typing = false
        continue_indicator.visible = true
    )

func _unhandled_input(event: InputEvent) -> void:
    if not visible:
        return
    if event.is_action_pressed("ui_accept"):
        if is_typing:
            # Skip typewriter
            text_label.visible_characters = -1
            is_typing = false
            continue_indicator.visible = true
        else:
            current_line += 1
            _display_line()
        get_viewport().set_input_as_handled()

func _close() -> void:
    visible = false
    dialog_finished.emit()
```

### Pause Menu

```gdscript
# pause_menu.gd
extends Control

func _ready() -> void:
    visible = false
    process_mode = Node.PROCESS_MODE_ALWAYS  # Process even when paused!

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("pause"):
        toggle_pause()
        get_viewport().set_input_as_handled()

func toggle_pause() -> void:
    visible = !visible
    get_tree().paused = visible
    
    if visible:
        Input.mouse_mode = Input.MOUSE_MODE_VISIBLE
        $VBox/ResumeButton.grab_focus()
    else:
        Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

func _on_resume_pressed() -> void:
    toggle_pause()

func _on_quit_pressed() -> void:
    get_tree().paused = false
    get_tree().change_scene_to_file("res://scenes/main_menu.tscn")
```

---

## Theme System

Themes centralize your entire UI's visual style — **don't use individual overrides for everything!**

### Creating a Theme

1. `FileSystem → New Resource → Theme` → save as `game_theme.tres`
2. Set as project default: `Project Settings → GUI → Theme → Custom`
3. Or assign to any Control node's `theme` property

### Theme Properties

Themes define 5 types of properties per Control type:

```
Colors     — font_color, font_pressed_color, etc.
Constants  — margin_left, separation, outline_size
Fonts      — font (the actual font resource)
Font Sizes — font_size
Icons      — checked, unchecked, radio_checked
Styles     — panel, normal, hover, pressed, disabled, focus
```

### StyleBoxFlat Example

```gdscript
# Create a button style programmatically
func create_button_style() -> StyleBoxFlat:
    var style := StyleBoxFlat.new()
    style.bg_color = Color("2d2d2d")
    style.border_color = Color("4a4a4a")
    style.set_border_width_all(2)
    style.set_corner_radius_all(8)
    style.set_content_margin_all(12)
    
    # Shadow
    style.shadow_color = Color(0, 0, 0, 0.3)
    style.shadow_size = 4
    style.shadow_offset = Vector2(2, 2)
    
    return style
```

### Applying Themes via Code

```gdscript
# Theme overrides on individual nodes (use sparingly)
label.add_theme_color_override("font_color", Color.RED)
label.add_theme_font_size_override("font_size", 24)
button.add_theme_stylebox_override("normal", custom_stylebox)

# Check if override exists
if label.has_theme_color_override("font_color"):
    pass

# Remove override
label.remove_theme_color_override("font_color")
```

---

## Theme Type Variations

Create variants of the same Control type for different UI contexts:

```
Base Button style:     Dark gray, white text
"DangerButton" variant: Red, white text, red border
"GhostButton" variant:  Transparent, white text, border only
```

In the Theme editor:
1. Add Type → `DangerButton`
2. Add style properties, inheriting from `Button`
3. On your Button node, set `Theme Type Variation` to `DangerButton`

---

## Responsive UI

### Screen Size Handling

```gdscript
# Project Settings → Display → Window
# - Size: Set your target resolution (1920x1080, 1280x720, etc.)
# - Stretch → Mode: "canvas_items" (scale UI) or "viewport" (pixel art)
# - Stretch → Aspect: "keep" (letterbox) or "expand" (fill)

# For pixel art games:
# Mode: viewport, Aspect: keep, Scale: integer

# For HD games:
# Mode: canvas_items, Aspect: expand
```

### Adapt to Window Resize

```gdscript
func _ready() -> void:
    get_viewport().size_changed.connect(_on_viewport_resized)

func _on_viewport_resized() -> void:
    var viewport_size := get_viewport_rect().size
    # Adjust UI layout based on viewport size
    if viewport_size.x < 800:
        # Mobile layout
        sidebar.visible = false
    else:
        # Desktop layout
        sidebar.visible = true
```

---

## UI Animation

### Menu Transitions

```gdscript
func slide_in_menu(menu: Control) -> void:
    menu.visible = true
    menu.modulate.a = 0.0
    menu.position.y += 50
    
    var tween := create_tween().set_parallel(true)
    tween.tween_property(menu, "modulate:a", 1.0, 0.3)
    tween.tween_property(menu, "position:y", menu.position.y - 50, 0.3)\
        .set_ease(Tween.EASE_OUT).set_trans(Tween.TRANS_CUBIC)

func slide_out_menu(menu: Control) -> void:
    var tween := create_tween().set_parallel(true)
    tween.tween_property(menu, "modulate:a", 0.0, 0.2)
    tween.tween_property(menu, "position:y", menu.position.y + 50, 0.2)\
        .set_ease(Tween.EASE_IN).set_trans(Tween.TRANS_CUBIC)
    
    tween.set_parallel(false)
    tween.tween_callback(func(): menu.visible = false)
```

### Staggered List Animation

```gdscript
func animate_list_items(container: VBoxContainer) -> void:
    for i in container.get_child_count():
        var child := container.get_child(i) as Control
        child.modulate.a = 0.0
        child.position.x -= 30
        
        var tween := create_tween()
        tween.tween_property(child, "modulate:a", 1.0, 0.2)\
            .set_delay(i * 0.05)
        tween.parallel().tween_property(child, "position:x", child.position.x + 30, 0.2)\
            .set_delay(i * 0.05)\
            .set_ease(Tween.EASE_OUT).set_trans(Tween.TRANS_CUBIC)
```

---

## UI Focus & Navigation

Critical for gamepad support:

```gdscript
# Set focus neighbors in code
button1.focus_neighbor_bottom = button2.get_path()
button2.focus_neighbor_top = button1.get_path()

# Or auto-calculate
func setup_focus_chain(buttons: Array[Button]) -> void:
    for i in buttons.size():
        var current := buttons[i]
        var next := buttons[(i + 1) % buttons.size()]
        var prev := buttons[(i - 1 + buttons.size()) % buttons.size()]
        current.focus_neighbor_bottom = next.get_path()
        current.focus_neighbor_top = prev.get_path()

# Grab focus (essential for keyboard/gamepad)
func _ready() -> void:
    $VBox/PlayButton.grab_focus()

# Focus visual feedback
# Set the "focus" StyleBox in your theme
```

---

## Practical Examples

### Settings Menu with Volume Sliders

```gdscript
extends Control

@onready var master_slider: HSlider = $VBox/MasterSlider
@onready var music_slider: HSlider = $VBox/MusicSlider
@onready var sfx_slider: HSlider = $VBox/SFXSlider
@onready var fullscreen_check: CheckBox = $VBox/FullscreenCheck

func _ready() -> void:
    # Load saved settings
    master_slider.value = db_to_linear(
        AudioServer.get_bus_volume_db(AudioServer.get_bus_index("Master"))
    )
    music_slider.value = db_to_linear(
        AudioServer.get_bus_volume_db(AudioServer.get_bus_index("Music"))
    )
    sfx_slider.value = db_to_linear(
        AudioServer.get_bus_volume_db(AudioServer.get_bus_index("SFX"))
    )
    fullscreen_check.button_pressed = (
        DisplayServer.window_get_mode() == DisplayServer.WINDOW_MODE_FULLSCREEN
    )
    
    master_slider.value_changed.connect(_on_master_changed)
    music_slider.value_changed.connect(_on_music_changed)
    sfx_slider.value_changed.connect(_on_sfx_changed)
    fullscreen_check.toggled.connect(_on_fullscreen_toggled)

func _on_master_changed(value: float) -> void:
    AudioServer.set_bus_volume_db(
        AudioServer.get_bus_index("Master"),
        linear_to_db(value)
    )

func _on_music_changed(value: float) -> void:
    AudioServer.set_bus_volume_db(
        AudioServer.get_bus_index("Music"),
        linear_to_db(value)
    )

func _on_sfx_changed(value: float) -> void:
    AudioServer.set_bus_volume_db(
        AudioServer.get_bus_index("SFX"),
        linear_to_db(value)
    )

func _on_fullscreen_toggled(toggled: bool) -> void:
    if toggled:
        DisplayServer.window_set_mode(DisplayServer.WINDOW_MODE_FULLSCREEN)
    else:
        DisplayServer.window_set_mode(DisplayServer.WINDOW_MODE_WINDOWED)
```

---

*← [04 — Animation System](./04-animation-system.md) | [06 — Shaders & VFX](./06-shaders-and-vfx.md) →*
