# 12 — Export & Publishing

> Get your game onto Steam, Android, iOS, Web, and more.

---

## Table of Contents

- [Export Basics](#export-basics)
- [Export Templates](#export-templates)
- [Platform: Windows](#windows)
- [Platform: Linux](#linux)
- [Platform: macOS](#macos)
- [Platform: Web (HTML5)](#web)
- [Platform: Android](#android)
- [Platform: iOS](#ios)
- [Platform: Steam](#steam)
- [Export Optimization](#export-optimization)
- [CI/CD Automation](#cicd)

---

## Export Basics

### Setting Up Exports

1. `Project → Export → Add...` → Choose platform
2. Configure settings
3. Click "Export Project" or "Export PCK/ZIP"

### Export Modes

```
Debug Export:
├── Includes debug symbols
├── Faster to build
├── Larger file size
└── Use for: Testing on target platform

Release Export:
├── Optimized and stripped
├── Slower to build
├── Smaller file size
└── Use for: Publishing / distribution
```

---

## Export Templates

Download export templates: `Editor → Manage Export Templates → Download`

Match your Godot version exactly! Godot 4.3 project needs 4.3 templates.

---

## Windows

```
Preset: Windows Desktop
├── Architecture: x86_64 (most PCs), x86_32 (legacy), arm64 (Surface Pro X)
├── Binary format: .exe
├── Icon: .ico file (256x256 recommended)
├── Product Name: "Your Game Name"
├── Company Name: "Your Studio"
├── File Description: "An amazing game"
├── Copyright: "© 2024 Your Studio"
├── Trademarks: ""
└── Embedded PCK: ✅ (single file distribution)

Advanced:
├── Modify Resources → enable for custom icon
├── Codesigning → sign with certificate for trust
└── Application manifest → DPI awareness settings
```

### Windows Tips

```
• Enable "Embed PCK" for single-.exe distribution
• Set DPI awareness for crisp rendering on high-DPI displays
• Code-sign your executable to prevent "Unknown Publisher" warnings
• Consider including VC++ redistributable or bundling with installer
```

---

## Linux

```
Preset: Linux
├── Architecture: x86_64, x86_32, arm64, arm32, rv64
├── Binary format: no extension (e.g., "my_game")
├── Embed PCK: ✅
└── Strip debug symbols: ✅ (for release)

Tips:
• Make the binary executable: chmod +x my_game
• Test on multiple distros (Ubuntu, Fedora, Arch)
• Bundle as AppImage or Flatpak for maximum compatibility
```

---

## macOS

```
Preset: macOS
├── Architecture: Universal (x86_64 + arm64) recommended
├── Application: .app bundle (actually a folder)
├── Codesigning: Required for notarization
├── Notarization: Required for distribution outside App Store
│   ├── Apple Developer account needed
│   ├── Hardened Runtime: ✅
│   └── xcrun notarytool for submission
└── App Store: Additional entitlements and provisioning profiles needed

Tips:
• Universal builds work on Intel AND Apple Silicon
• Without notarization, users see "app can't be opened" 
• Distribute as .dmg for best UX
```

---

## Web

```
Preset: Web
├── Output: .html + .js + .wasm + .pck
├── VRAM Texture Compression: For 3D (ETC2 / S3TC)
├── Export Type: Regular (recommended)
├── Threads: SharedArrayBuffer (requires secure context headers)
└── Progressive Web App: Enable for installable web app

Hosting Requirements:
├── HTTPS required for SharedArrayBuffer (threads)
├── Server must support .wasm MIME type
├── Cross-Origin headers needed for threading:
│   Cross-Origin-Opener-Policy: same-origin
│   Cross-Origin-Embedder-Policy: require-corp
└── Use Godot's HTML template or customize

Tips:
• Web builds use Compatibility renderer (GL), not Forward+
• Audio starts only after user interaction (browser policy)
• File system access limited — use user:// for saves
• Test with a local HTTP server, not file:// URLs
```

### Quick Local Testing

```bash
# Python HTTP server with COOP/COEP headers
python -m http.server 8080
# Then open localhost:8080 in browser
```

---

## Android

```
Prerequisites:
├── Android SDK (install via Android Studio)
├── Java JDK 17+
├── Android Debug Keystore (auto-generated)
└── Set paths in Editor Settings → Export → Android

Preset: Android
├── Package name: com.yourstudio.yourgame
├── Min SDK: 24 (Android 7.0) — recommended minimum
├── Target SDK: 34 (latest)
├── Permissions: Configure per your game's needs
│   ├── INTERNET (if multiplayer)
│   ├── VIBRATE (if haptics)
│   └── WRITE_EXTERNAL_STORAGE (older APIs, saves)
├── Screen orientation: Landscape / Portrait / Sensor
├── Graphics: Vulkan (Mobile) or GL Compatibility
├── Keystore: Debug for testing, Release for publishing
└── APK vs AAB: Use AAB for Google Play Store

Tips:
• Google Play requires AAB (Android App Bundle) format
• Test on multiple screen sizes/densities
• Use GL Compatibility for maximum device support
• Touch input mapped to mouse events by default
```

---

## iOS

```
Prerequisites:
├── macOS computer (required!)
├── Xcode installed
├── Apple Developer Account ($99/year)
└── Valid provisioning profiles and certificates

Preset: iOS
├── Bundle Identifier: com.yourstudio.yourgame
├── Team ID: From Apple Developer portal
├── App Store Icon: 1024x1024 PNG
├── Launch Screen: Storyboard or images
├── Capabilities: Game Center, Push Notifications, etc.
├── Architectures: arm64
└── Export generates Xcode project → build in Xcode

Tips:
• Must build final IPA in Xcode on a Mac
• TestFlight for beta testing
• App Store review typically 1-3 days
• iOS specific: handle safe area insets for notch devices
```

---

## Steam

### Setup

1. Create a Steamworks developer account ($100 one-time)
2. Set up app on Steamworks dashboard
3. Get your App ID

### Integration with GodotSteam

```
• Plugin: GodotSteam (https://github.com/GodotSteam/GodotSteam)
• Provides: Achievements, Leaderboards, Cloud Saves, Networking,
  Workshop, Rich Presence, Overlay, DLC checking
• Installation: Download matching build or compile from source
```

```gdscript
# steam_manager.gd (Autoload)
extends Node

var steam_running: bool = false

func _ready() -> void:
    var initialize := Steam.steamInitEx(false, YOUR_APP_ID)
    if initialize.status != Steam.STEAM_API_INIT_RESULT_OK:
        push_warning("Steam init failed: %s" % initialize.verbal)
        return
    
    steam_running = true
    print("Steam initialized! User: %s" % Steam.getPersonaName())

func _process(_delta: float) -> void:
    if steam_running:
        Steam.run_callbacks()

# Achievements
func unlock_achievement(achievement_id: String) -> void:
    if steam_running:
        Steam.setAchievement(achievement_id)
        Steam.storeStats()

# Cloud saves
func save_to_cloud(filename: String, data: PackedByteArray) -> bool:
    if steam_running:
        return Steam.fileWrite(filename, data)
    return false
```

### Steam Deployment

```
1. Upload builds using SteamCMD or Steamworks GUI
2. Configure depots (one per platform)
3. Set launch options for each platform
4. Configure store page, screenshots, descriptions
5. Request store review
6. Publish!

Structure:
├── depot_windows/     → Windows build
├── depot_linux/       → Linux build
├── depot_mac/         → macOS build
└── steamcmd_scripts/  → Upload automation
```

---

## Export Optimization

### Reduce File Size

```
1. Compress textures (VRAM compression in import settings)
2. Remove unused files (use "Detect Unused Resources" plugin)  
3. Compress audio (OGG instead of WAV for music)
4. Strip debug symbols (release export)
5. Split large PCK files (for downloadable content)
6. Exclude folders: .git, docs/, test/

# Custom export filters:
# In Export preset → Resources → Filters to Export:
# Include: *.tscn, *.gd, *.tres, *.png, *.ogg, *.wav
# Exclude: *.md, *.txt, *.sample
```

### Platform-Specific Optimization

```
Mobile:
├── Use GL Compatibility renderer
├── Reduce texture sizes (1024x1024 max)
├── Limit particle counts
├── Disable expensive post-processing
└── Target 30fps if needed

Web:
├── Use GL Compatibility renderer
├── Minimize download size
├── Preload only essential assets
├── Defer loading non-essential content
└── Consider disabling threads for broader browser support
```

---

## CI/CD

### GitHub Actions for Godot

```yaml
# .github/workflows/export.yml
name: Export Godot Project
on:
  push:
    tags: ["v*"]

jobs:
  export:
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3
    steps:
      - uses: actions/checkout@v4
      
      - name: Export Windows
        run: |
          mkdir -p build/windows
          godot --headless --export-release "Windows Desktop" build/windows/game.exe
      
      - name: Export Linux
        run: |
          mkdir -p build/linux
          godot --headless --export-release "Linux" build/linux/game
      
      - name: Export Web
        run: |
          mkdir -p build/web
          godot --headless --export-release "Web" build/web/index.html
      
      - name: Upload Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: game-builds
          path: build/
```

---

*← [11 — Save & Load](./11-save-and-load.md) | [13 — GDExtension](./13-gdextension.md) →*
