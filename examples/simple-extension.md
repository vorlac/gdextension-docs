# Simple Extension Example

**What this is:** A complete, minimal GDExtension that demonstrates the essential concepts of creating a C++ class usable in Godot. This example shows how to create a custom Node2D class that moves in a sine wave pattern, exposes properties to the editor, and handles user interaction.

**What it demonstrates:** Basic class registration, property binding, method exposure to GDScript, signal emission, and the complete build process. This is the starting point for understanding how GDExtensions work before moving to more complex patterns.

**When to use this pattern:** Use this as a template for simple gameplay components, basic utility classes, or when learning GDExtension development. Perfect for single-file extensions that don't need complex architecture.

## Project Structure

```
simple_extension/
├── src/
│   ├── register_types.cpp
│   ├── register_types.h
│   ├── simple_node.cpp
│   └── simple_node.h
├── demo/
│   ├── project.godot
│   ├── main.tscn
│   └── bin/
│       └── simple_extension.gdextension
├── SConstruct
└── README.md
```

## Source Files

### src/simple_node.h

```cpp
#ifndef SIMPLE_NODE_H
#define SIMPLE_NODE_H

#include <godot_cpp/classes/node2d.hpp>
#include <godot_cpp/core/class_db.hpp>

namespace godot {

class SimpleNode : public Node2D {
    GDCLASS(SimpleNode, Node2D)

private:
    double amplitude = 10.0;
    double frequency = 1.0;
    double time_passed = 0.0;
    Vector2 initial_position;
    bool is_enabled = true;
    String custom_text = "Hello from GDExtension!";
    int click_count = 0;

protected:
    static void _bind_methods();
    void _notification(int p_what);

public:
    SimpleNode();
    ~SimpleNode();

    // Property setters/getters
    void set_amplitude(double p_amplitude);
    double get_amplitude() const;

    void set_frequency(double p_frequency);
    double get_frequency() const;

    void set_enabled(bool p_enabled);
    bool is_enabled() const;

    void set_custom_text(const String &p_text);
    String get_custom_text() const;

    int get_click_count() const;

    // Methods
    void reset_position();
    void print_debug_info();
    Vector2 calculate_offset(double p_time) const;

    // Virtual methods
    virtual void _ready() override;
    virtual void _process(double p_delta) override;
    virtual void _input(const Ref<InputEvent> &p_event) override;
};

}

#endif // SIMPLE_NODE_H
```

### src/simple_node.cpp

```cpp
#include "simple_node.h"

#include <godot_cpp/core/class_db.hpp>
#include <godot_cpp/variant/utility_functions.hpp>
#include <godot_cpp/classes/input_event_mouse_button.hpp>

using namespace godot;

void SimpleNode::_bind_methods() {
    // Bind methods
    ClassDB::bind_method(D_METHOD("reset_position"), &SimpleNode::reset_position);
    ClassDB::bind_method(D_METHOD("print_debug_info"), &SimpleNode::print_debug_info);
    ClassDB::bind_method(D_METHOD("calculate_offset", "time"), &SimpleNode::calculate_offset);
    ClassDB::bind_method(D_METHOD("get_click_count"), &SimpleNode::get_click_count);

    // Bind properties
    ClassDB::bind_method(D_METHOD("set_amplitude", "amplitude"), &SimpleNode::set_amplitude);
    ClassDB::bind_method(D_METHOD("get_amplitude"), &SimpleNode::get_amplitude);
    ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "amplitude", PROPERTY_HINT_RANGE, "0,100,0.1"),
                 "set_amplitude", "get_amplitude");

    ClassDB::bind_method(D_METHOD("set_frequency", "frequency"), &SimpleNode::set_frequency);
    ClassDB::bind_method(D_METHOD("get_frequency"), &SimpleNode::get_frequency);
    ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "frequency", PROPERTY_HINT_RANGE, "0,10,0.1"),
                 "set_frequency", "get_frequency");

    ClassDB::bind_method(D_METHOD("set_enabled", "enabled"), &SimpleNode::set_enabled);
    ClassDB::bind_method(D_METHOD("is_enabled"), &SimpleNode::is_enabled);
    ADD_PROPERTY(PropertyInfo(Variant::BOOL, "enabled"), "set_enabled", "is_enabled");

    ClassDB::bind_method(D_METHOD("set_custom_text", "text"), &SimpleNode::set_custom_text);
    ClassDB::bind_method(D_METHOD("get_custom_text"), &SimpleNode::get_custom_text);
    ADD_PROPERTY(PropertyInfo(Variant::STRING, "custom_text"), "set_custom_text", "get_custom_text");

    // Bind signals
    ADD_SIGNAL(MethodInfo("position_reset"));
    ADD_SIGNAL(MethodInfo("clicked", PropertyInfo(Variant::INT, "count")));

    // Bind constants
    BIND_CONSTANT(10);  // Example constant
}

SimpleNode::SimpleNode() {
    UtilityFunctions::print("SimpleNode constructor called");
}

SimpleNode::~SimpleNode() {
    UtilityFunctions::print("SimpleNode destructor called");
}

void SimpleNode::_notification(int p_what) {
    switch (p_what) {
        case NOTIFICATION_ENTER_TREE:
            UtilityFunctions::print("SimpleNode entered the tree");
            break;
        case NOTIFICATION_EXIT_TREE:
            UtilityFunctions::print("SimpleNode exited the tree");
            break;
    }
}

void SimpleNode::_ready() {
    UtilityFunctions::print(custom_text);
    initial_position = get_position();
}

void SimpleNode::_process(double p_delta) {
    if (!is_enabled) {
        return;
    }

    time_passed += p_delta;

    Vector2 offset = calculate_offset(time_passed);
    set_position(initial_position + offset);
}

void SimpleNode::_input(const Ref<InputEvent> &p_event) {
    const InputEventMouseButton *mouse_event = Object::cast_to<InputEventMouseButton>(p_event.ptr());

    if (mouse_event && mouse_event->is_pressed() && mouse_event->get_button_index() == MOUSE_BUTTON_LEFT) {
        Vector2 mouse_pos = mouse_event->get_position();
        Vector2 local_pos = get_global_transform().affine_inverse().xform(mouse_pos);

        // Simple hit detection (assuming a 100x100 area)
        if (local_pos.x >= -50 && local_pos.x <= 50 &&
            local_pos.y >= -50 && local_pos.y <= 50) {
            click_count++;
            emit_signal("clicked", click_count);
            UtilityFunctions::print("SimpleNode clicked! Count: ", click_count);
        }
    }
}

void SimpleNode::set_amplitude(double p_amplitude) {
    amplitude = p_amplitude;
}

double SimpleNode::get_amplitude() const {
    return amplitude;
}

void SimpleNode::set_frequency(double p_frequency) {
    frequency = p_frequency;
}

double SimpleNode::get_frequency() const {
    return frequency;
}

void SimpleNode::set_enabled(bool p_enabled) {
    is_enabled = p_enabled;
    if (!is_enabled) {
        reset_position();
    }
}

bool SimpleNode::is_enabled() const {
    return is_enabled;
}

void SimpleNode::set_custom_text(const String &p_text) {
    custom_text = p_text;
}

String SimpleNode::get_custom_text() const {
    return custom_text;
}

int SimpleNode::get_click_count() const {
    return click_count;
}

void SimpleNode::reset_position() {
    set_position(initial_position);
    time_passed = 0.0;
    emit_signal("position_reset");
}

void SimpleNode::print_debug_info() {
    UtilityFunctions::print("=== SimpleNode Debug Info ===");
    UtilityFunctions::print("Position: ", get_position());
    UtilityFunctions::print("Amplitude: ", amplitude);
    UtilityFunctions::print("Frequency: ", frequency);
    UtilityFunctions::print("Time Passed: ", time_passed);
    UtilityFunctions::print("Enabled: ", is_enabled);
    UtilityFunctions::print("Click Count: ", click_count);
}

Vector2 SimpleNode::calculate_offset(double p_time) const {
    double x = amplitude * Math::sin(frequency * p_time * Math_TAU);
    double y = amplitude * Math::cos(frequency * p_time * Math_TAU);
    return Vector2(x, y);
}
```

### src/register_types.h

```cpp
#ifndef SIMPLE_EXTENSION_REGISTER_TYPES_H
#define SIMPLE_EXTENSION_REGISTER_TYPES_H

#include <godot_cpp/core/class_db.hpp>

using namespace godot;

void initialize_simple_extension_module(ModuleInitializationLevel p_level);
void uninitialize_simple_extension_module(ModuleInitializationLevel p_level);

#endif // SIMPLE_EXTENSION_REGISTER_TYPES_H
```

### src/register_types.cpp

```cpp
#include "register_types.h"

#include <gdextension_interface.h>
#include <godot_cpp/core/defs.hpp>
#include <godot_cpp/godot.hpp>

#include "simple_node.h"

using namespace godot;

void initialize_simple_extension_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }

    // Register our custom class
    GDREGISTER_CLASS(SimpleNode);

    UtilityFunctions::print("Simple Extension initialized successfully!");
}

void uninitialize_simple_extension_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }

    UtilityFunctions::print("Simple Extension uninitialized");
}

extern "C" {
// Initialization
GDExtensionBool GDE_EXPORT simple_extension_init(
    GDExtensionInterfaceGetProcAddress p_get_proc_address,
    GDExtensionClassLibraryPtr p_library,
    GDExtensionInitialization *r_initialization
) {
    godot::GDExtensionBinding::InitObject init_obj(p_get_proc_address, p_library, r_initialization);

    init_obj.register_initializer(initialize_simple_extension_module);
    init_obj.register_terminator(uninitialize_simple_extension_module);
    init_obj.set_minimum_library_initialization_level(MODULE_INITIALIZATION_LEVEL_SCENE);

    return init_obj.init();
}
}
```

## Build Configuration

### SConstruct

```python
#!/usr/bin/env python
import os

# Detect platform
def get_platform():
    from sys import platform
    if platform == "win32":
        return "windows"
    elif platform == "darwin":
        return "macos"
    elif platform == "linux":
        return "linux"
    else:
        return "unknown"

env = SConscript("../godot-cpp/SConstruct")

# Add source files
env.Append(CPPPATH=["src/"])
sources = Glob("src/*.cpp")

# Platform-specific settings
platform = get_platform()
if platform == "windows":
    library_name = "simple_extension.windows.{}.{}".format(
        env["target"], env["arch"])
    library = env.SharedLibrary(
        "demo/bin/lib{}".format(library_name),
        source=sources
    )
elif platform == "macos":
    library_name = "simple_extension.macos.{}.{}".format(
        env["target"], env["arch"])
    library = env.SharedLibrary(
        "demo/bin/lib{}".format(library_name),
        source=sources
    )
elif platform == "linux":
    library_name = "simple_extension.linux.{}.{}".format(
        env["target"], env["arch"])
    library = env.SharedLibrary(
        "demo/bin/lib{}".format(library_name),
        source=sources
    )

Default(library)
```

## Godot Configuration

### demo/bin/simple_extension.gdextension

```ini
[configuration]

entry_symbol = "simple_extension_init"
compatibility_minimum = "4.3"
reloadable = true

[libraries]

macos.debug = "res://bin/libsimple_extension.macos.template_debug.universal.dylib"
macos.release = "res://bin/libsimple_extension.macos.template_release.universal.dylib"

windows.debug.x86_64 = "res://bin/libsimple_extension.windows.template_debug.x86_64.dll"
windows.release.x86_64 = "res://bin/libsimple_extension.windows.template_release.x86_64.dll"

linux.debug.x86_64 = "res://bin/libsimple_extension.linux.template_debug.x86_64.so"
linux.release.x86_64 = "res://bin/libsimple_extension.linux.template_release.x86_64.so"

android.debug.arm64 = "res://bin/libsimple_extension.android.template_debug.arm64.so"
android.release.arm64 = "res://bin/libsimple_extension.android.template_release.arm64.so"

[dependencies]

macos = {}
windows = {}
linux = {}
android = {}
```

### demo/project.godot

```ini
; Engine configuration file.

[application]

config/name="Simple Extension Demo"
config/features=PackedStringArray("4.3", "GL Compatibility")
run/main_scene="res://main.tscn"

[rendering]

renderer/rendering_method="gl_compatibility"
renderer/rendering_method.mobile="gl_compatibility"
```

## Usage Example (GDScript)

### demo/test_simple_node.gd

```gdscript
extends Node

func _ready():
    # Create SimpleNode instance
    var simple_node = SimpleNode.new()
    simple_node.name = "MySimpleNode"
    simple_node.position = Vector2(400, 300)
    add_child(simple_node)

    # Configure properties
    simple_node.amplitude = 50.0
    simple_node.frequency = 2.0
    simple_node.custom_text = "Configured from GDScript!"

    # Connect signals
    simple_node.position_reset.connect(_on_position_reset)
    simple_node.clicked.connect(_on_node_clicked)

    # Call methods
    simple_node.print_debug_info()

    # Test calculate_offset
    var offset = simple_node.calculate_offset(1.0)
    print("Calculated offset: ", offset)

func _on_position_reset():
    print("Position was reset!")

func _on_node_clicked(count: int):
    print("Node clicked ", count, " times!")

func _input(event):
    if event.is_action_pressed("ui_accept"):
        var simple_node = get_node("MySimpleNode")
        if simple_node:
            simple_node.reset_position()
```

## Building the Extension

```bash
# Clone godot-cpp
git clone https://github.com/godotengine/godot-cpp.git
cd godot-cpp
git checkout 4.3

# Build godot-cpp
scons platform=<platform> target=template_debug
scons platform=<platform> target=template_release

# Build the extension
cd ../simple_extension
scons platform=<platform> target=template_debug
scons platform=<platform> target=template_release

# Run Godot
cd demo
godot --editor
```

## Key Concepts Demonstrated

1. **Class Registration**: Using `GDCLASS` macro and `GDREGISTER_CLASS`
2. **Property Binding**: Exposing properties to Godot with hints
3. **Method Binding**: Making C++ methods callable from GDScript
4. **Signal Definition**: Creating and emitting custom signals
5. **Virtual Methods**: Overriding `_ready()`, `_process()`, `_input()`
6. **Type Casting**: Using `Object::cast_to<>` for safe casting
7. **Utility Functions**: Using `UtilityFunctions::print()` for debugging
8. **Math Functions**: Using `Math::sin()`, `Math::cos()`, constants
9. **Input Handling**: Processing mouse events
10. **Property Hints**: Using `PROPERTY_HINT_RANGE` for editor UI

## Summary

This simple extension demonstrates:
- **~200 lines** of functional code
- **Complete integration** with Godot's node system
- **Property system** with editor integration
- **Signal system** for event handling
- **Input processing** with hit detection
- **Animation** using process callback
- **Debug utilities** for development

The example provides a solid foundation for understanding GDExtension development.
