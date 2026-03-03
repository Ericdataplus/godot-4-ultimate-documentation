# 18 — Project Structure & Organization

> How to organize your Godot project so it scales from prototype to shipped game.

---

## Recommended Folder Structure

```
res://
├── assets/
│   ├── audio/
│   │   ├── music/
│   │   ├── sfx/
│   │   └── voice/
│   ├── fonts/
│   ├── sprites/
│   │   ├── characters/
│   │   ├── environment/
│   │   ├── ui/
│   │   └── effects/
│   ├── models/        (3D projects)
│   ├── textures/      (3D projects)
│   ├── shaders/
│   └── themes/
│
├── scenes/
│   ├── characters/
│   │   ├── player/
│   │   │   ├── player.tscn
│   │   │   └── player.gd
│   │   └── enemies/
│   │       ├── goblin.tscn
│   │       ├── goblin.gd
│   │       └── slime.tscn
│   ├── levels/
│   │   ├── level_01.tscn
│   │   ├── level_02.tscn
│   │   └── level_boss.tscn
│   ├── ui/
│   │   ├── hud.tscn
│   │   ├── main_menu.tscn
│   │   ├── pause_menu.tscn
│   │   └── settings_menu.tscn
│   ├── pickups/
│   │   ├── coin.tscn
│   │   └── health_potion.tscn
│   └── vfx/
│       ├── explosion.tscn
│       └── hit_particles.tscn
│
├── scripts/
│   ├── autoloads/
│   │   ├── game_manager.gd
│   │   ├── audio_manager.gd
│   │   ├── save_manager.gd
│   │   └── event_bus.gd
│   ├── components/
│   │   ├── health_component.gd
│   │   ├── hitbox.gd
│   │   ├── hurtbox.gd
│   │   ├── velocity_component.gd
│   │   └── state_machine.gd
│   ├── resources/
│   │   ├── weapon_resource.gd
│   │   ├── item_resource.gd
│   │   └── enemy_data.gd
│   └── utils/
│       ├── constants.gd
│       └── helpers.gd
│
├── data/
│   ├── weapons/
│   │   ├── sword.tres
│   │   ├── bow.tres
│   │   └── staff.tres
│   ├── enemies/
│   │   ├── goblin_data.tres
│   │   └── slime_data.tres
│   └── items/
│       ├── health_potion.tres
│       └── mana_potion.tres
│
├── addons/           (Third-party plugins)
│
├── default_bus_layout.tres
├── project.godot
├── export_presets.cfg
├── .gitignore
└── .gitattributes
```

---

## Naming Conventions

```
Files & Folders:    snake_case       → player.gd, main_menu.tscn
Nodes:              PascalCase       → Player, HealthBar, EnemySpawner
Classes:            PascalCase       → class_name PlayerController
Functions:          snake_case       → func take_damage()
Variables:          snake_case       → var move_speed
Constants:          UPPER_SNAKE_CASE → const MAX_SPEED
Signals:            snake_case       → signal health_changed
Enums:              PascalCase       → enum State { IDLE, WALKING }
```

---

## Git Integration

### .gitignore for Godot

```gitignore
# Godot-specific
.godot/
*.uid

# Imported resources (regenerated)
# Don't ignore .import folder — it's needed!

# OS-specific
.DS_Store
Thumbs.db
*.tmp

# Build outputs
export/
build/

# IDE
.vscode/
*.code-workspace
```

### .gitattributes

```gitattributes
# Godot resource files — treat as text
*.tscn text
*.tres text
*.godot text

# Binary assets — use LFS
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.ttf filter=lfs diff=lfs merge=lfs -text
*.otf filter=lfs diff=lfs merge=lfs -text
*.glb filter=lfs diff=lfs merge=lfs -text
*.gltf filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text
```

---

## When to Split Scenes

```
Split into sub-scenes when:
✅ Node tree has 15+ nodes
✅ Logic can be reused elsewhere
✅ Multiple developers need to edit simultaneously (merge conflicts!)
✅ The component is logically independent (HitBox, HurtBox, HealthBar)

Keep in one scene when:
✅ Nodes are tightly coupled (always change together)
✅ Fewer than 10 nodes total
✅ Breaking apart adds complexity without benefit
```

---

## File Organization Rules

```
1. ONE SCRIPT PER FILE (no multi-class files)
2. Script sits NEXT TO its scene (player.gd next to player.tscn)
3. Resources (.tres) go in data/ organized by type
4. Autoloads go in scripts/autoloads/
5. Shared components go in scripts/components/
6. Don't put scripts in the root directory
7. Use descriptive names (not "test.gd" or "new_script.gd")
```

---

*← [17 — Tips & Tricks](./17-tips-and-tricks.md) | [19 — Modular Architecture](./19-modular-architecture.md) →*
