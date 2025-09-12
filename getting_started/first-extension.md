# First Extension Tutorial - Step-by-Step Guide

## Introduction

**Building your first GDExtension from scratch:** This tutorial walks you through creating a complete GDExtension, from setting up your development environment to using your custom class in Godot. We'll build a rotating sprite controller that demonstrates core concepts like property binding, signal emission, and virtual method implementation.

## Prerequisites

Before starting, ensure you have:
- Completed the [Quick Start Guide](quick-start.md)
- Basic C++ knowledge (classes, pointers, inheritance)
- Godot 4.3+ installed
- A C++ development environment set up

## Project Setup

### Step 1: Create Project Structure

Create the following directory structure:

```
rotating-sprite/
├── godot-cpp/              # Will contain godot-cpp as submodule
├── src/
│   ├── register_types.cpp
│   ├── register_types.h
│   ├── rotating_sprite.cpp
│   └── rotating_sprite.h
├── demo/                   # Godot project for testing
│   └── project.godot
├── SConstruct
└── rotating_sprite.gdextension
```

### Step 2: Add godot-cpp as Submodule

```bash
cd rotating-sprite
git init
git submodule add https://github.com/godotengine/godot-cpp.git
cd godot-cpp
git checkout 4.3  # or master for latest
cd ..
```

### Step 3: Create the Build Script

Create `SConstruct`:

```python
#!/usr/bin/env python
import os
import sys

# Add godot-cpp build system
env = SConscript("godot-cpp/SConstruct")

# Add our source paths
env.Append(CPPPATH=["src/"])
sources = Glob("src/*.cpp")

# Determine the output file name based on platform
if env["platform"] == "macos":
    library = env.SharedLibrary(
        "demo/bin/librotating_sprite.{}.{}.framework/librotating_sprite.{}.{}".format(
            env["platform"], env["target"], env["platform"], env["target"]
        ),
        source=sources,
    )
elif env["platform"] == "windows":
    library = env.SharedLibrary(
        "demo/bin/librotating_sprite.{}.{}.{}".format(
            env["platform"], env["target"], env["arch"]
        ),
        source=sources,
    )
else:
    library = env.SharedLibrary(
        "demo/bin/librotating_sprite.{}.{}.{}".format(
            env["platform"], env["target"], env["arch"]
        ),
        source=sources,
    )

Default(library)
```

## Writing the Extension Code

### Step 4: Create the Header File

Create `src/rotating_sprite.h`:

```cpp
#ifndef ROTATING_SPRITE_H
#define ROTATING_SPRITE_H

#include <godot_cpp/classes/sprite2d.hpp>
#include <godot_cpp/core/class_db.hpp>

using namespace godot;

class RotatingSprite : public Sprite2D {
    GDCLASS(RotatingSprite, Sprite2D)

private:
    double time_passed;
    double rotation_speed;
    bool auto_rotate;
    Vector2 orbit_center;
    float orbit_radius;
    
protected:
    static void _bind_methods();
    
    // Notification handler
    void _notification(int p_what);
    
public:
    RotatingSprite();
    ~RotatingSprite();
    
    // Virtual method overrides
    void _ready() override;
    void _process(double delta) override;
    
    // Custom methods
    void start_rotation();
    void stop_rotation();
    void reset_rotation();
    Vector2 calculate_orbit_position(double angle) const;
    
    // Property setters/getters
    void set_rotation_speed(double p_speed);
    double get_rotation_speed() const;
    
    void set_auto_rotate(bool p_enabled);
    bool is_auto_rotating() const;
    
    void set_orbit_radius(float p_radius);
    float get_orbit_radius() const;
    
    void set_orbit_center(const Vector2 &p_center);
    Vector2 get_orbit_center() const;
    
    // Constants
    static constexpr double DEFAULT_SPEED = 1.0;
    static constexpr float DEFAULT_ORBIT_RADIUS = 100.0f;
};

#endif // ROTATING_SPRITE_H
```

### Step 5: Implement the Class

Create `src/rotating_sprite.cpp`:

```cpp
#include "rotating_sprite.h"
#include <godot_cpp/core/class_db.hpp>
#include <godot_cpp/variant/utility_functions.hpp>

using namespace godot;

void RotatingSprite::_bind_methods() {
    // Bind methods
    ClassDB::bind_method(D_METHOD("start_rotation"), &RotatingSprite::start_rotation);
    ClassDB::bind_method(D_METHOD("stop_rotation"), &RotatingSprite::stop_rotation);
    ClassDB::bind_method(D_METHOD("reset_rotation"), &RotatingSprite::reset_rotation);
    ClassDB::bind_method(D_METHOD("calculate_orbit_position", "angle"), 
                        &RotatingSprite::calculate_orbit_position);
    
    // Bind properties
    ClassDB::bind_method(D_METHOD("set_rotation_speed", "speed"), 
                        &RotatingSprite::set_rotation_speed);
    ClassDB::bind_method(D_METHOD("get_rotation_speed"), 
                        &RotatingSprite::get_rotation_speed);
    ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "rotation_speed", 
                             PROPERTY_HINT_RANGE, "0.0,10.0,0.1"), 
                "set_rotation_speed", "get_rotation_speed");
    
    ClassDB::bind_method(D_METHOD("set_auto_rotate", "enabled"), 
                        &RotatingSprite::set_auto_rotate);
    ClassDB::bind_method(D_METHOD("is_auto_rotating"), 
                        &RotatingSprite::is_auto_rotating);
    ADD_PROPERTY(PropertyInfo(Variant::BOOL, "auto_rotate"), 
                "set_auto_rotate", "is_auto_rotating");
    
    ClassDB::bind_method(D_METHOD("set_orbit_radius", "radius"), 
                        &RotatingSprite::set_orbit_radius);
    ClassDB::bind_method(D_METHOD("get_orbit_radius"), 
                        &RotatingSprite::get_orbit_radius);
    ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "orbit_radius", 
                             PROPERTY_HINT_RANGE, "0.0,500.0,1.0"), 
                "set_orbit_radius", "get_orbit_radius");
    
    ClassDB::bind_method(D_METHOD("set_orbit_center", "center"), 
                        &RotatingSprite::set_orbit_center);
    ClassDB::bind_method(D_METHOD("get_orbit_center"), 
                        &RotatingSprite::get_orbit_center);
    ADD_PROPERTY(PropertyInfo(Variant::VECTOR2, "orbit_center"), 
                "set_orbit_center", "get_orbit_center");
    
    // Add property group
    ADD_GROUP("Rotation Settings", "");
    
    // Bind signals
    ADD_SIGNAL(MethodInfo("rotation_started"));
    ADD_SIGNAL(MethodInfo("rotation_stopped"));
    ADD_SIGNAL(MethodInfo("rotation_completed", 
                         PropertyInfo(Variant::INT, "rotations")));
    
    // Bind constants
    BIND_CONSTANT(DEFAULT_SPEED);
    BIND_CONSTANT(DEFAULT_ORBIT_RADIUS);
}

RotatingSprite::RotatingSprite() {
    time_passed = 0.0;
    rotation_speed = DEFAULT_SPEED;
    auto_rotate = false;
    orbit_center = Vector2(0, 0);
    orbit_radius = DEFAULT_ORBIT_RADIUS;
    
    UtilityFunctions::print("RotatingSprite created!");
}

RotatingSprite::~RotatingSprite() {
    UtilityFunctions::print("RotatingSprite destroyed!");
}

void RotatingSprite::_notification(int p_what) {
    switch (p_what) {
        case NOTIFICATION_ENTER_TREE:
            UtilityFunctions::print("RotatingSprite entered tree");
            break;
        case NOTIFICATION_EXIT_TREE:
            UtilityFunctions::print("RotatingSprite exited tree");
            break;
        case NOTIFICATION_READY:
            // Called when _ready() is triggered
            break;
        case NOTIFICATION_PROCESS:
            // Called when _process() is triggered
            break;
    }
}

void RotatingSprite::_ready() {
    UtilityFunctions::print("RotatingSprite is ready!");
    
    // Store initial position as orbit center if not set
    if (orbit_center == Vector2(0, 0)) {
        orbit_center = get_position();
    }
    
    if (auto_rotate) {
        start_rotation();
    }
}

void RotatingSprite::_process(double delta) {
    if (!auto_rotate) {
        return;
    }
    
    time_passed += delta;
    
    // Rotate the sprite
    set_rotation(time_passed * rotation_speed);
    
    // Orbit around center
    if (orbit_radius > 0) {
        Vector2 new_pos = calculate_orbit_position(time_passed * rotation_speed);
        set_position(new_pos);
    }
    
    // Emit signal every full rotation
    static int last_rotation = 0;
    int current_rotation = static_cast<int>(time_passed * rotation_speed / (2 * Math_PI));
    if (current_rotation > last_rotation) {
        emit_signal("rotation_completed", current_rotation);
        last_rotation = current_rotation;
    }
}

void RotatingSprite::start_rotation() {
    if (!auto_rotate) {
        auto_rotate = true;
        emit_signal("rotation_started");
        UtilityFunctions::print("Rotation started");
    }
}

void RotatingSprite::stop_rotation() {
    if (auto_rotate) {
        auto_rotate = false;
        emit_signal("rotation_stopped");
        UtilityFunctions::print("Rotation stopped");
    }
}

void RotatingSprite::reset_rotation() {
    time_passed = 0.0;
    set_rotation(0.0);
    set_position(orbit_center);
    UtilityFunctions::print("Rotation reset");
}

Vector2 RotatingSprite::calculate_orbit_position(double angle) const {
    float x = orbit_center.x + cos(angle) * orbit_radius;
    float y = orbit_center.y + sin(angle) * orbit_radius;
    return Vector2(x, y);
}

void RotatingSprite::set_rotation_speed(double p_speed) {
    rotation_speed = p_speed;
}

double RotatingSprite::get_rotation_speed() const {
    return rotation_speed;
}

void RotatingSprite::set_auto_rotate(bool p_enabled) {
    auto_rotate = p_enabled;
    if (auto_rotate) {
        start_rotation();
    } else {
        stop_rotation();
    }
}

bool RotatingSprite::is_auto_rotating() const {
    return auto_rotate;
}

void RotatingSprite::set_orbit_radius(float p_radius) {
    orbit_radius = MAX(0.0f, p_radius);
}

float RotatingSprite::get_orbit_radius() const {
    return orbit_radius;
}

void RotatingSprite::set_orbit_center(const Vector2 &p_center) {
    orbit_center = p_center;
}

Vector2 RotatingSprite::get_orbit_center() const {
    return orbit_center;
}
```

### Step 6: Create Registration Files

Create `src/register_types.h`:

```cpp
#ifndef ROTATING_SPRITE_REGISTER_TYPES_H
#define ROTATING_SPRITE_REGISTER_TYPES_H

#include <godot_cpp/core/class_db.hpp>

using namespace godot;

void initialize_rotating_sprite_module(ModuleInitializationLevel p_level);
void uninitialize_rotating_sprite_module(ModuleInitializationLevel p_level);

#endif // ROTATING_SPRITE_REGISTER_TYPES_H
```

Create `src/register_types.cpp`:

```cpp
#include "register_types.h"

#include <gdextension_interface.h>
#include <godot_cpp/core/defs.hpp>
#include <godot_cpp/godot.hpp>

#include "rotating_sprite.h"

using namespace godot;

void initialize_rotating_sprite_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }

    GDREGISTER_CLASS(RotatingSprite);
    
    UtilityFunctions::print("RotatingSprite extension initialized!");
}

void uninitialize_rotating_sprite_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }
    
    UtilityFunctions::print("RotatingSprite extension uninitialized!");
}

extern "C" {
// Initialization
GDExtensionBool GDE_EXPORT rotating_sprite_library_init(
    GDExtensionInterfaceGetProcAddress p_get_proc_address,
    GDExtensionClassLibraryPtr p_library,
    GDExtensionInitialization *r_initialization) {
    
    godot::GDExtensionBinding::InitObject init_obj(p_get_proc_address, p_library, r_initialization);

    init_obj.register_initializer(initialize_rotating_sprite_module);
    init_obj.register_terminator(uninitialize_rotating_sprite_module);
    init_obj.set_minimum_library_initialization_level(MODULE_INITIALIZATION_LEVEL_SCENE);

    return init_obj.init();
}
}
```

### Step 7: Configure the Extension

Create `rotating_sprite.gdextension`:

```ini
[configuration]
entry_symbol = "rotating_sprite_library_init"
compatibility_minimum = "4.3"
reloadable = true

[libraries]
macos.debug = "demo/bin/librotating_sprite.macos.template_debug.framework"
macos.release = "demo/bin/librotating_sprite.macos.template_release.framework"
windows.debug.x86_32 = "demo/bin/librotating_sprite.windows.template_debug.x86_32.dll"
windows.release.x86_32 = "demo/bin/librotating_sprite.windows.template_release.x86_32.dll"
windows.debug.x86_64 = "demo/bin/librotating_sprite.windows.template_debug.x86_64.dll"
windows.release.x86_64 = "demo/bin/librotating_sprite.windows.template_release.x86_64.dll"
linux.debug.x86_64 = "demo/bin/librotating_sprite.linux.template_debug.x86_64.so"
linux.release.x86_64 = "demo/bin/librotating_sprite.linux.template_release.x86_64.so"
linux.debug.arm64 = "demo/bin/librotating_sprite.linux.template_debug.arm64.so"
linux.release.arm64 = "demo/bin/librotating_sprite.linux.template_release.arm64.so"
```

## Building the Extension

### Step 8: Compile

```bash
# Build godot-cpp first
cd godot-cpp
scons platform=<your_platform> target=template_debug
cd ..

# Build your extension
scons platform=<your_platform> target=template_debug

# For release build
scons platform=<your_platform> target=template_release
```

Replace `<your_platform>` with `windows`, `linux`, or `macos`.

## Testing in Godot

### Step 9: Create Test Project

1. Create `demo/project.godot`:

```ini
; Engine configuration file.
; It's best edited using the editor UI and not directly,
; since the properties are organized in sections here.

[application]

config/name="RotatingSprite Demo"
config/features=PackedStringArray("4.3")

[rendering]

renderer/rendering_method="forward_plus"
```

2. Copy `rotating_sprite.gdextension` to the `demo/` folder

3. Open the project in Godot

### Step 10: Use Your Extension

Create a test scene in Godot:

1. Add a RotatingSprite node (it will appear in the "Add Node" dialog under Sprite2D)
2. Assign a texture to the sprite
3. In the Inspector, you'll see your custom properties:
   - Rotation Speed
   - Auto Rotate
   - Orbit Radius
   - Orbit Center

Create a GDScript to interact with your extension:

```gdscript
extends Node

@onready var rotating_sprite = $RotatingSprite

func _ready():
    # Connect to signals
    rotating_sprite.rotation_started.connect(_on_rotation_started)
    rotating_sprite.rotation_stopped.connect(_on_rotation_stopped)
    rotating_sprite.rotation_completed.connect(_on_rotation_completed)
    
    # Configure properties
    rotating_sprite.rotation_speed = 2.0
    rotating_sprite.orbit_radius = 150.0
    rotating_sprite.auto_rotate = true

func _on_rotation_started():
    print("Rotation started!")

func _on_rotation_stopped():
    print("Rotation stopped!")

func _on_rotation_completed(rotations):
    print("Completed rotation #", rotations)

func _input(event):
    if event.is_action_pressed("ui_accept"):
        if rotating_sprite.is_auto_rotating():
            rotating_sprite.stop_rotation()
        else:
            rotating_sprite.start_rotation()
    elif event.is_action_pressed("ui_cancel"):
        rotating_sprite.reset_rotation()
```

## Understanding What You Built

### Key Concepts Demonstrated

1. **Class Registration**: Your class is registered with Godot's ClassDB
2. **Property Binding**: Properties appear in the Inspector
3. **Method Binding**: Methods callable from GDScript
4. **Signal Emission**: Custom signals for event communication
5. **Virtual Methods**: Overriding `_ready()` and `_process()`
6. **Notifications**: Handling engine notifications
7. **Constants**: Exposing constants to GDScript

### File Structure Explained

- **register_types.cpp**: Entry point and class registration
- **rotating_sprite.h/cpp**: Your custom class implementation
- **SConstruct**: Build configuration
- **.gdextension**: Maps compiled libraries to platforms

## Common Issues and Solutions

### Build Errors

**Issue**: "Cannot find godot_cpp headers"
```bash
# Solution: Initialize submodules
git submodule update --init --recursive
```

**Issue**: "Undefined symbols"
```bash
# Solution: Ensure godot-cpp is built first
cd godot-cpp && scons platform=<platform> && cd ..
```

### Runtime Errors

**Issue**: "Extension not loading"
- Check .gdextension file paths
- Verify library names match build output
- Ensure compatibility_minimum matches your Godot version

**Issue**: "Class not appearing in editor"
- Verify GDREGISTER_CLASS is called
- Check initialization level is correct
- Ensure class inherits from a Godot class

## Next Steps

Now that you've built your first extension:

1. **Explore More Features**:
   - Add more complex properties (Arrays, Resources)
   - Implement tool scripts (editor functionality)
   - Create custom resources

2. **Learn Advanced Topics**:
   - [Memory Management](../architecture/memory-model.md)
   - [Signal Patterns](../guides/signal-patterns.md)
   - [Performance Optimization](../advanced-topics/performance-optimization.md)

3. **Study Examples**:
   - [Simple Extension](../examples/simple-extension.md)
   - [Advanced Extension](../examples/advanced-extension.md)

## Conclusion

Congratulations! You've created a fully functional GDExtension that:
- Extends existing Godot classes
- Adds custom properties and methods
- Emits signals
- Integrates seamlessly with the editor
- Can be scripted from GDScript

This foundation prepares you for building more complex extensions. Continue to [Basic Concepts](basic-concepts.md) to deepen your understanding of GDExtension architecture.