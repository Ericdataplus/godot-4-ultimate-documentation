# 15 — Editor Secrets & Shortcuts

> Hidden features, keyboard shortcuts, and workflow hacks that make you 10x faster.

---

## Essential Shortcuts

### Scene Editor

| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Add new node |
| `Ctrl+Shift+A` | Instantiate scene |
| `Ctrl+D` | Duplicate selected node |
| `Ctrl+Shift+D` | Duplicate and move (works in 2D/3D) |
| `Delete` | Delete selected node |
| `Ctrl+C / V` | Copy / Paste nodes |
| `F` | Focus camera on selected (2D & 3D) |
| `Ctrl+G` | Group selected nodes |
| `F5` | Run project |
| `F6` | Run current scene |
| `F8` | Stop running |
| `Ctrl+Shift+F11` | Distraction-free mode |
| `Ctrl+Z / Y` | Undo / Redo |

### Script Editor

| Shortcut | Action |
|----------|--------|
| `Ctrl+Space` | Autocomplete |
| `Ctrl+Click` | Go to definition |
| `Ctrl+Shift+F` | Search across all files |
| `Ctrl+K` | Toggle comment |
| `Ctrl+Shift+K` | Uncomment block |
| `Alt+Up/Down` | Move line up/down |
| `Ctrl+Shift+Up/Down` | Duplicate line |
| `Ctrl+D` | Select next occurrence |
| `Alt+Click` | Add cursor (multi-cursor!) |
| `Ctrl+L` | Go to line |
| `Ctrl+R` | Find and replace |
| `Ctrl+Shift+R` | Replace in files |
| `F12` | Go to declaration |
| `Ctrl+I` | Auto-indent selection |

### 2D Viewport

| Shortcut | Action |
|----------|--------|
| `Q` | Select mode |
| `W` | Move mode |
| `E` | Rotate mode |
| `S` | Scale mode |
| `V` | Pivot mode |
| `Hold Ctrl` | Temporarily disable snapping |
| `Hold Shift` | Constrain to axis |
| `Middle click drag` | Pan viewport |
| `Scroll` | Zoom |
| `Double-click zoom` | Reset zoom to 100% |

### 3D Viewport

| Shortcut | Action |
|----------|--------|
| `Right-click + WASD` | Free-flight mode (hold) |
| `Scroll in free-flight` | Adjust fly speed |
| `Numpad 1/3/7` | Front/Right/Top orthographic |
| `Numpad 5` | Toggle perspective/orthographic |
| `Numpad .` | Focus on selected |
| `Shift+F` | Align camera with view |
| `Ctrl+F` | Align view with camera |

---

## Hidden Editor Features

### Drag & Drop Tricks

```
• Drag a file from FileSystem into script → auto-generates preload()
• Hold Ctrl + drag a file into script → generates preload() with path
• Drag a node from SceneTree into script → generates get_node() path
• Hold Ctrl + drag node into script → generates @onready var declaration
• Drag a node from SceneTree into FileSystem → saves as new scene
```

### Inspector Tricks

```
• Math in numeric fields: Type "100*2+50" and press Enter = 250
• Arrow keys adjust values (Ctrl = ×100, Shift = ×10, Alt = ×0.1)
• Right-click property → Copy Property Path (for animations & code)
• Right-click node → "Change Type" to switch node type without losing children
• Right-click instanced scene → "Editable Children" for modifications
• Click the key 🔑 icon next to any property to add animation keyframe
```

### Scene Tree Tricks

```
• Shift+Click on collapse arrows → Recursive collapse/uncollapse
• Right-click node → "Save Branch as Scene" → Extract sub-tree
• Right-click in 2D viewport → "Instantiate Scene Here" / "Add Node Here"
• Select node, right-click viewport → "Move Nodes Here"
• F2 → Rename node
• Click % icon → make node accessible with unique name (%NodeName)
```

### Unique Name Access (%)

```gdscript
# In the SceneTree, right-click a node → "Access as Unique Name"
# A % icon appears. Now access from ANYWHERE in the scene:

@onready var health_bar: ProgressBar = %HealthBar
# No more fragile paths like $UI/HUD/Bars/HealthBar!

# Works regardless of where the node is in the tree
# As long as it's in the same scene
```

### Remote Debugging

```
While the game is running:
1. Click "Remote" tab in SceneTree
2. See live game state
3. Change properties in real-time!
4. Changes are NOT saved to source

Project Settings → Debug:
• Show collision shapes in running game
• Show navigation meshes
• Show paths
```

### Editor Settings Worth Changing

```
Editor → Editor Settings:
• Text Editor → Behavior → Scroll Past End of File → ✅
• Text Editor → Completion → Code Autocomplete → ✅
• Text Editor → Appearance → Word Wrap → ✅
• Interface → Theme → Preset → "Default" or "Custom"
• Interface → Editor → Custom Display Scale (for HiDPI)
• Debug → Visual → Show Collision Shapes When Running → ✅

• Inspector → Default Color Picker Wheel → Choose your style
• FileSystem → Show Hidden Files → ✅ (for .gitignore etc.)
```

### Custom Editor Layouts

```
Editor → Editor Layout → Save Layout → name it
Create different layouts for:
• "Scripting" — large script panel
• "Level Design" — large 2D/3D viewport
• "Animation" — large animation panel
• "Debugging" — profiler + output visible
```

### Print Debugging

```gdscript
# Pretty-print scene tree
print_tree()
print_tree_pretty()

# Rich text output with colors
print_rich("[color=red]Error![/color] Player health: [b]%d[/b]" % health)

# Debug draw (2D)
func _draw() -> void:
    draw_circle(Vector2.ZERO, 50, Color.RED)
    draw_line(Vector2.ZERO, Vector2(100, 0), Color.BLUE, 2.0)
    draw_rect(Rect2(-25, -25, 50, 50), Color.GREEN, false, 2.0)

# Force redraw in _process
queue_redraw()

# Debug draw (3D)
# Use DebugDraw3D addon or draw with ImmediateMesh
```

### Project Manager Tags

```
In the Project Manager:
• Click "Edit" on a project → add tags for organization
• Set a custom icon (square PNG) as project thumbnail
• Tags help filter when you have many projects
```

---

## Workflow Hacks

### Play Scene Shortcuts

```
F5         → Play main scene
F6         → Play current scene
F7         → Pause running game
F8         → Stop running game

Right-click scene tab → "Play This Scene"
(saves clicking through menus)

Project Settings → "Main Scene" to set the default F5 scene
```

### Scene Tab Management

```
• Middle-click a tab to close it
• Drag tabs to reorder
• "Run This Scene" from tab context menu
```

### File System Favorites

```
• Right-click folders → "Add to Favorites"
• Favorites appear at the top of the FileSystem dock
• Essential for large projects
```

### Godot Command Palette

```
Ctrl+Shift+P → Command palette (search for any editor action)
Like VS Code's command palette!
```

---

*← [14 — Performance Bible](./14-performance-bible.md) | [16 — Common Pitfalls](./16-common-pitfalls.md) →*
