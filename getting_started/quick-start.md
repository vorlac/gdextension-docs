# Quick Start Guide

## Prerequisites

Before starting with godot-cpp, ensure you have:

- **Godot Engine 4.3+** installed
- **C++ compiler** (MSVC 2019+, GCC 9+, or Clang 10+)
- **Python 3.6+** for build scripts
- **SCons** build system (`pip install scons`)
- **Git** for version control

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/godotengine/godot-cpp.git
cd godot-cpp
git submodule update --init --recursive
```

### 2. Build the Bindings

```bash
# Build debug version (default)
scons

# Build release version
scons target=template_release

# Platform-specific builds
scons platform=windows  # or linux, macos, android, ios, web
```

## Your First Extension

### 1. Create Project Structure

```
my_extension/
├── src/
│   └── my_node.cpp
├── SConstruct
└── my_extension.gdextension
```

### 2. Write Your C++ Code

```cpp
// src/my_node.cpp
#include <godot_cpp/classes/node2d.hpp>
#include <godot_cpp/core/class_db.hpp>
#include <godot_cpp/godot.hpp>

using namespace godot;

class MyNode : public Node2D {
    GDCLASS(MyNode, Node2D)

private:
    double time_passed = 0.0;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("get_time"), &MyNode::get_time);
    }

public:
    void _process(double delta) override {
        time_passed += delta;
        set_position(Vector2(100.0 * sin(time_passed), 100.0 * cos(time_passed)));
    }

    double get_time() const { return time_passed; }
};

// Module initialization
extern "C" {
GDExtensionBool GDE_EXPORT my_extension_init(
    GDExtensionInterfaceGetProcAddress p_get_proc_address,
    GDExtensionClassLibraryPtr p_library,
    GDExtensionInitialization *r_initialization) {

    godot::GDExtensionBinding::InitObject init_obj(p_get_proc_address, p_library, r_initialization);

    init_obj.register_initializer([](ModuleInitializationLevel p_level) {
        if (p_level == MODULE_INITIALIZATION_LEVEL_SCENE) {
            ClassDB::register_class<MyNode>();
        }
    });

    return init_obj.init();
}
}
```

### 3. Create Build Script

```python
# SConstruct
#!/usr/bin/env python
import os

env = SConscript("godot-cpp/SConstruct")

# Add your source files
env.Append(CPPPATH=["src/"])
sources = Glob("src/*.cpp")

# Build the library
if env["platform"] == "windows":
    library = env.SharedLibrary(
        "bin/my_extension.{}".format(env["platform"]),
        source=sources,
    )
else:
    library = env.SharedLibrary(
        "bin/libmy_extension.{}.{}".format(env["platform"], env["arch"]),
        source=sources,
    )

Default(library)
```

### 4. Configure Extension

```ini
# my_extension.gdextension
[configuration]
entry_symbol = "my_extension_init"
compatibility_minimum = "4.3"

[libraries]
windows.debug.x86_64 = "bin/my_extension.windows.template_debug.x86_64.dll"
windows.release.x86_64 = "bin/my_extension.windows.template_release.x86_64.dll"
linux.debug.x86_64 = "bin/libmy_extension.linux.template_debug.x86_64.so"
linux.release.x86_64 = "bin/libmy_extension.linux.template_release.x86_64.so"
macos.debug = "bin/libmy_extension.macos.template_debug.framework"
macos.release = "bin/libmy_extension.macos.template_release.framework"
```

### 5. Build and Use

```bash
# Build your extension
scons

# Copy to Godot project
cp -r bin/ /path/to/godot/project/
cp my_extension.gdextension /path/to/godot/project/
```

In Godot:
1. Open your project
2. The extension loads automatically
3. Add your custom node from "Add Node" dialog
4. Your `MyNode` will appear under Node2D

## Next Steps

- [First Extension Tutorial](first-extension.md) - Detailed walkthrough
- [Basic Concepts](basic-concepts.md) - Understanding GDExtension
- [Core Architecture](../architecture/core-architecture.md) - System design
- [Simple Extension Example](../examples/simple-extension.md) - Complete example

## Common Issues

### Build Errors

```bash
# Missing godot-cpp
git submodule update --init --recursive

# Wrong platform
scons platform=windows  # Match your OS

# Missing dependencies
pip install scons
```

### Runtime Errors

- **Extension not loading**: Check .gdextension paths
- **Class not found**: Ensure GDREGISTER_CLASS is called
- **Crashes**: Check pointer validity and memory management

## Resources

- [Official Documentation](https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/)
- [API Reference](../api-reference/)
- [Examples](../examples/)
- [Community Discord](https://discord.gg/godotengine)
