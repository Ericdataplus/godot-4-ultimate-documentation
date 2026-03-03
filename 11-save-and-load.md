# 11 — Save & Load Systems

> Three approaches to game persistence: Resources, JSON, and ConfigFile.

---

## Table of Contents

- [Approach Comparison](#approach-comparison)
- [Method 1: Custom Resources (Recommended)](#custom-resources)
- [Method 2: JSON Files](#json-files)
- [Method 3: ConfigFile](#configfile)
- [Encryption](#encryption)
- [Save File Migration](#migration)
- [Auto-Save Pattern](#auto-save)

---

## Approach Comparison

| Method | Best For | Pros | Cons |
|--------|----------|------|------|
| **Custom Resource** | Complex game state | Type-safe, native Godot integration | Can execute code on load (security) |
| **JSON** | Simple data, web games | Human-readable, cross-platform safe | No type safety, manual serialization |
| **ConfigFile** | Settings only | INI-like, simple key-value | Not great for complex nested data |

---

## Custom Resources

**The recommended approach** for most games:

### Define Save Data Structure

```gdscript
# save_data.gd
extends Resource
class_name SaveData

@export var player_name: String = "Hero"
@export var level: int = 1
@export var experience: int = 0
@export var health: int = 100
@export var max_health: int = 100
@export var position: Vector2 = Vector2.ZERO
@export var current_scene: String = "res://scenes/levels/level_1.tscn"
@export var inventory: Array[ItemSaveData] = []
@export var unlocked_abilities: Array[String] = []
@export var play_time_seconds: float = 0.0
@export var save_timestamp: String = ""
@export var version: int = 1  # For migration
```

```gdscript
# item_save_data.gd
extends Resource
class_name ItemSaveData

@export var item_id: String = ""
@export var quantity: int = 1
@export var slot_index: int = -1
```

### Save & Load

```gdscript
# save_manager.gd (Autoload)
extends Node

const SAVE_DIR := "user://saves/"
const SAVE_EXTENSION := ".tres"

func _ready() -> void:
    DirAccess.make_dir_recursive_absolute(SAVE_DIR)

func save_game(slot: int = 0) -> bool:
    var data := SaveData.new()
    
    # Gather data from game
    var player := get_tree().get_first_node_in_group("player")
    if player:
        data.player_name = player.player_name
        data.health = player.health
        data.max_health = player.max_health
        data.position = player.global_position
    
    data.current_scene = get_tree().current_scene.scene_file_path
    data.save_timestamp = Time.get_datetime_string_from_system()
    data.play_time_seconds = GameManager.play_time
    
    # Save inventory
    for item in InventoryManager.items:
        var item_data := ItemSaveData.new()
        item_data.item_id = item.id
        item_data.quantity = item.quantity
        data.inventory.append(item_data)
    
    # Write to file
    var path := SAVE_DIR + "save_%d%s" % [slot, SAVE_EXTENSION]
    var error := ResourceSaver.save(data, path)
    
    if error != OK:
        push_error("Failed to save: %s" % error)
        return false
    
    print("Game saved to slot %d" % slot)
    return true

func load_game(slot: int = 0) -> bool:
    var path := SAVE_DIR + "save_%d%s" % [slot, SAVE_EXTENSION]
    
    if not FileAccess.file_exists(path):
        push_warning("No save file in slot %d" % slot)
        return false
    
    var data := ResourceLoader.load(path) as SaveData
    if data == null:
        push_error("Failed to load save data")
        return false
    
    # Apply data to game
    await get_tree().change_scene_to_file(data.current_scene)
    await get_tree().process_frame
    
    var player := get_tree().get_first_node_in_group("player")
    if player:
        player.player_name = data.player_name
        player.health = data.health
        player.max_health = data.max_health
        player.global_position = data.position
    
    # Restore inventory
    InventoryManager.clear()
    for item_data in data.inventory:
        InventoryManager.add_item(item_data.item_id, item_data.quantity)
    
    GameManager.play_time = data.play_time_seconds
    
    return true

func has_save(slot: int) -> bool:
    return FileAccess.file_exists(SAVE_DIR + "save_%d%s" % [slot, SAVE_EXTENSION])

func delete_save(slot: int) -> void:
    var path := SAVE_DIR + "save_%d%s" % [slot, SAVE_EXTENSION]
    if FileAccess.file_exists(path):
        DirAccess.remove_absolute(path)

func get_save_info(slot: int) -> Dictionary:
    var path := SAVE_DIR + "save_%d%s" % [slot, SAVE_EXTENSION]
    if not FileAccess.file_exists(path):
        return {}
    var data := ResourceLoader.load(path) as SaveData
    return {
        "name": data.player_name,
        "level": data.level,
        "timestamp": data.save_timestamp,
        "play_time": data.play_time_seconds,
    }
```

> **⚠️ Security Warning:** `ResourceLoader.load()` can execute GDScript! For user-facing save files you don't trust, use JSON instead.

---

## JSON Files

Safer for web games and when security matters:

```gdscript
# json_save_manager.gd
extends Node

const SAVE_PATH := "user://save.json"

func save_game() -> void:
    var data := {
        "version": 1,
        "player": {
            "name": player.player_name,
            "health": player.health,
            "position": {
                "x": player.global_position.x,
                "y": player.global_position.y,
            },
        },
        "inventory": [],
        "timestamp": Time.get_datetime_string_from_system(),
    }
    
    # Serialize inventory
    for item in inventory:
        data["inventory"].append({
            "id": item.id,
            "quantity": item.quantity,
        })
    
    var json_string := JSON.stringify(data, "\t")  # Pretty print
    var file := FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    file.store_string(json_string)
    file.close()

func load_game() -> Dictionary:
    if not FileAccess.file_exists(SAVE_PATH):
        return {}
    
    var file := FileAccess.open(SAVE_PATH, FileAccess.READ)
    var json_string := file.get_as_text()
    file.close()
    
    var json := JSON.new()
    var error := json.parse(json_string)
    if error != OK:
        push_error("JSON parse error: %s at line %d" % [json.get_error_message(), json.get_error_line()])
        return {}
    
    return json.data
```

### Saveable Interface Pattern

```gdscript
# Make any node saveable by implementing these methods:

# saveable.gd (no class, just a convention)
# Nodes that can be saved implement:
#   func save_data() -> Dictionary
#   func load_data(data: Dictionary) -> void

# Then iterate all saveables:
func gather_save_data() -> Dictionary:
    var data := {}
    for node in get_tree().get_nodes_in_group("saveable"):
        data[node.get_path()] = node.save_data()
    return data

func apply_save_data(data: Dictionary) -> void:
    for path in data:
        var node := get_node_or_null(path)
        if node and node.has_method("load_data"):
            node.load_data(data[path])
```

---

## ConfigFile

Best for settings (video, audio, controls):

```gdscript
# settings_manager.gd (Autoload)
extends Node

const SETTINGS_PATH := "user://settings.cfg"

var config := ConfigFile.new()

func _ready() -> void:
    load_settings()

func load_settings() -> void:
    var err := config.load(SETTINGS_PATH)
    if err != OK:
        # First run — set defaults
        _set_defaults()
        save_settings()

func save_settings() -> void:
    config.save(SETTINGS_PATH)

func _set_defaults() -> void:
    config.set_value("audio", "master_volume", 1.0)
    config.set_value("audio", "music_volume", 0.8)
    config.set_value("audio", "sfx_volume", 1.0)
    config.set_value("video", "fullscreen", false)
    config.set_value("video", "vsync", true)
    config.set_value("video", "resolution", "1920x1080")
    config.set_value("gameplay", "language", "en")
    config.set_value("gameplay", "screen_shake", true)

# Getters with defaults
func get_master_volume() -> float:
    return config.get_value("audio", "master_volume", 1.0)

func get_fullscreen() -> bool:
    return config.get_value("video", "fullscreen", false)

# Setters that auto-save
func set_master_volume(value: float) -> void:
    config.set_value("audio", "master_volume", value)
    save_settings()
    AudioServer.set_bus_volume_db(0, linear_to_db(value))
```

---

## Encryption

### Encrypted Save Files

```gdscript
const ENCRYPTION_KEY := "your-secret-key-here-change-this"

func save_encrypted(data: Dictionary) -> void:
    var json := JSON.stringify(data)
    var file := FileAccess.open_encrypted_with_pass(
        SAVE_PATH, FileAccess.WRITE, ENCRYPTION_KEY
    )
    file.store_string(json)
    file.close()

func load_encrypted() -> Dictionary:
    var file := FileAccess.open_encrypted_with_pass(
        SAVE_PATH, FileAccess.READ, ENCRYPTION_KEY
    )
    if file == null:
        return {}
    var json_string := file.get_as_text()
    file.close()
    
    var json := JSON.new()
    json.parse(json_string)
    return json.data
```

---

## Migration

Handle save file version upgrades:

```gdscript
func migrate_save(data: Dictionary) -> Dictionary:
    var version: int = data.get("version", 0)
    
    # v0 → v1: Added inventory system
    if version < 1:
        data["inventory"] = []
        data["version"] = 1
    
    # v1 → v2: Changed health to float
    if version < 2:
        data["player"]["health"] = float(data["player"].get("health", 100))
        data["player"]["max_health"] = float(data["player"].get("max_health", 100))
        data["version"] = 2
    
    # v2 → v3: Added quest system
    if version < 3:
        data["quests"] = {"active": [], "completed": []}
        data["version"] = 3
    
    return data
```

---

## Auto-Save

```gdscript
# auto_save.gd (add to your game scene)
extends Timer

@export var save_interval: float = 300.0  # 5 minutes
@export var save_slot: int = 99  # Dedicated auto-save slot

func _ready() -> void:
    wait_time = save_interval
    timeout.connect(_on_timeout)
    start()

func _on_timeout() -> void:
    SaveManager.save_game(save_slot)
    print("Auto-saved!")

# Also save on quit
func _notification(what: int) -> void:
    if what == NOTIFICATION_WM_CLOSE_REQUEST:
        SaveManager.save_game(save_slot)
        get_tree().quit()
```

---

*← [10 — Networking](./10-networking.md) | [12 — Export & Publishing](./12-export-and-publishing.md) →*
