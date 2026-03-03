# 13 — GDExtension & C++ Plugins

> Extend Godot's engine with C++ for maximum performance and custom node types.

---

## Table of Contents

- [When to Use GDExtension](#when-to-use-gdextension)
- [Setup](#setup)
- [Creating Your First Extension](#first-extension)
- [Binding Classes](#binding-classes)
- [Calling from GDScript](#calling-from-gdscript)
- [Build System](#build-system)
- [Debugging](#debugging)
- [Tips](#tips)

---

## When to Use GDExtension

```
USE GDExtension when:
✅ Performance-critical code (pathfinding, procedural generation, physics)
✅ Interfacing with C/C++ libraries (Steam, Discord, custom engines)
✅ Creating custom node types with native performance
✅ Porting existing C++ codebases to Godot
✅ Implementing algorithms that GDScript can't handle fast enough

DON'T use GDExtension when:
❌ GDScript is fast enough (90% of game logic)
❌ Simple gameplay scripts
❌ Prototype / game jam projects
❌ You don't know C++ well
```

---

## Setup

### Prerequisites

```
1. C++ compiler (MSVC on Windows, Clang on macOS, GCC on Linux)
2. SCons build system: pip install scons
3. Python 3.6+
4. godot-cpp (git submodule or download)
```

### Project Structure

```
your_extension/
├── godot-cpp/              ← Git submodule
│   └── (Godot C++ bindings)
├── src/
│   ├── register_types.h
│   ├── register_types.cpp
│   ├── my_node.h
│   └── my_node.cpp
├── demo/                   ← Godot project to test
│   ├── project.godot
│   ├── bin/
│   │   └── my_extension.gdextension
│   └── test_scene.tscn
├── SConstruct
└── .gitmodules
```

### Initialize

```bash
# Clone your project
mkdir my_extension && cd my_extension

# Add godot-cpp as submodule
git submodule add https://github.com/godotengine/godot-cpp.git
cd godot-cpp
git checkout godot-4.3-stable  # Match your Godot version!
cd ..

# Generate bindings
cd godot-cpp
scons platform=<your_platform> custom_api_file=../gdextension_api.json
cd ..
```

---

## First Extension

### register_types.h

```cpp
#ifndef REGISTER_TYPES_H
#define REGISTER_TYPES_H

#include <godot_cpp/core/class_db.hpp>

using namespace godot;

void initialize_my_extension(ModuleInitializationLevel p_level);
void uninitialize_my_extension(ModuleInitializationLevel p_level);

#endif
```

### register_types.cpp

```cpp
#include "register_types.h"
#include "my_node.h"

#include <gdextension_interface.h>
#include <godot_cpp/core/defs.hpp>
#include <godot_cpp/godot.hpp>

using namespace godot;

void initialize_my_extension(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }
    ClassDB::register_class<MyNode>();
}

void uninitialize_my_extension(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }
}

extern "C" {
GDExtensionBool GDE_EXPORT my_extension_init(
    GDExtensionInterfaceGetProcAddress p_get_proc_address,
    const GDExtensionClassLibraryPtr p_library,
    GDExtensionInitialization *r_initialization
) {
    godot::GDExtensionBinding::InitObject init_obj(
        p_get_proc_address, p_library, r_initialization
    );

    init_obj.register_initializer(initialize_my_extension);
    init_obj.register_terminator(uninitialize_my_extension);
    init_obj.set_minimum_library_initialization_level(
        MODULE_INITIALIZATION_LEVEL_SCENE
    );

    return init_obj.init();
}
}
```

---

## Binding Classes

### my_node.h

```cpp
#ifndef MY_NODE_H
#define MY_NODE_H

#include <godot_cpp/classes/node2d.hpp>
#include <godot_cpp/core/class_db.hpp>

namespace godot {

class MyNode : public Node2D {
    GDCLASS(MyNode, Node2D)

private:
    float speed;
    int health;

protected:
    static void _bind_methods();

public:
    MyNode();
    ~MyNode();

    void set_speed(float p_speed);
    float get_speed() const;

    void set_health(int p_health);
    int get_health() const;

    void take_damage(int amount);

    void _process(double delta) override;
    void _ready() override;
};

} // namespace godot

#endif
```

### my_node.cpp

```cpp
#include "my_node.h"

#include <godot_cpp/core/class_db.hpp>
#include <godot_cpp/variant/utility_functions.hpp>

using namespace godot;

void MyNode::_bind_methods() {
    // Properties (show in Inspector)
    ClassDB::bind_method(D_METHOD("set_speed", "speed"), &MyNode::set_speed);
    ClassDB::bind_method(D_METHOD("get_speed"), &MyNode::get_speed);
    ADD_PROPERTY(
        PropertyInfo(Variant::FLOAT, "speed", PROPERTY_HINT_RANGE, "0,500,0.1"),
        "set_speed", "get_speed"
    );

    ClassDB::bind_method(D_METHOD("set_health", "health"), &MyNode::set_health);
    ClassDB::bind_method(D_METHOD("get_health"), &MyNode::get_health);
    ADD_PROPERTY(
        PropertyInfo(Variant::INT, "health", PROPERTY_HINT_RANGE, "0,1000,1"),
        "set_health", "get_health"
    );

    // Regular methods (callable from GDScript)
    ClassDB::bind_method(D_METHOD("take_damage", "amount"), &MyNode::take_damage);

    // Signals
    ADD_SIGNAL(MethodInfo("health_changed",
        PropertyInfo(Variant::INT, "new_health"),
        PropertyInfo(Variant::INT, "max_health")
    ));
    ADD_SIGNAL(MethodInfo("died"));
}

MyNode::MyNode() {
    speed = 200.0f;
    health = 100;
}

MyNode::~MyNode() {}

void MyNode::set_speed(float p_speed) { speed = p_speed; }
float MyNode::get_speed() const { return speed; }

void MyNode::set_health(int p_health) {
    health = p_health;
    emit_signal("health_changed", health, 100);
}
int MyNode::get_health() const { return health; }

void MyNode::take_damage(int amount) {
    health -= amount;
    emit_signal("health_changed", health, 100);
    if (health <= 0) {
        emit_signal("died");
    }
}

void MyNode::_ready() {
    UtilityFunctions::print("MyNode is ready! Speed: ", speed);
}

void MyNode::_process(double delta) {
    // Fast C++ processing here
}
```

---

## Calling from GDScript

```gdscript
# Once built and loaded, MyNode appears in the Add Node dialog!
# Use it just like any built-in node:

var my_node := MyNode.new()
my_node.speed = 300.0
my_node.health = 50
my_node.take_damage(10)
my_node.health_changed.connect(_on_health_changed)
```

---

## Build System

### SConstruct

```python
#!/usr/bin/env python
import os, sys

env = SConscript("godot-cpp/SConstruct")

# Add source files
env.Append(CPPPATH=["src/"])
sources = Glob("src/*.cpp")

# Build the library
library = env.SharedLibrary(
    "demo/bin/libmy_extension{}{}".format(env["suffix"], env["SHLIBSUFFIX"]),
    source=sources,
)

Default(library)
```

### Build Commands

```bash
# Windows (MSVC)
scons platform=windows

# Linux
scons platform=linux

# macOS
scons platform=macos

# Debug build
scons platform=windows target=template_debug

# Release build
scons platform=windows target=template_release
```

### .gdextension File

```ini
# demo/bin/my_extension.gdextension
[configuration]
entry_symbol = "my_extension_init"
compatibility_minimum = "4.3"

[libraries]
windows.debug.x86_64 = "res://bin/libmy_extension.windows.template_debug.x86_64.dll"
windows.release.x86_64 = "res://bin/libmy_extension.windows.template_release.x86_64.dll"
linux.debug.x86_64 = "res://bin/libmy_extension.linux.template_debug.x86_64.so"
linux.release.x86_64 = "res://bin/libmy_extension.linux.template_release.x86_64.so"
macos.debug = "res://bin/libmy_extension.macos.template_debug.framework"
macos.release = "res://bin/libmy_extension.macos.template_release.framework"
```

---

## Debugging

```
• Build with target=template_debug
• Attach your C++ debugger (Visual Studio, GDB, LLDB)
• Set breakpoints in your C++ code
• Run Godot from the debugger
• UtilityFunctions::print() outputs to Godot's console
• Use UtilityFunctions::push_error() for errors
```

---

## Tips

```
1. Always match godot-cpp version to your Godot version
2. Hot-reload works! Change code, rebuild, Godot auto-reloads
3. Start simple — bind one class, test, then expand
4. Use C++ for compute-heavy work, GDScript for game logic
5. Export your extension .dll/.so with your game
6. Consider Rust (gdext) as an alternative to C++
```

---

*← [12 — Export & Publishing](./12-export-and-publishing.md) | [14 — Performance Bible](./14-performance-bible.md) →*
