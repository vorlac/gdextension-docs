# Frequently Asked Questions

## Getting Started

### Q: What's the difference between GDExtension and GDNative?
**A:** GDExtension is the newer, improved system that replaced GDNative in Godot 4.0. Key improvements include:
- Better performance with direct function calls instead of going through a C API layer
- Improved memory management and reduced overhead
- Better debugging support and error reporting
- Simplified build process and project setup
- Hot-reloading capabilities for faster development

### Q: Which version of godot-cpp should I use?
**A:** Use the branch that matches your Godot version:
- `master` branch tracks Godot's development version
- `4.4` branch for Godot 4.4.x stable releases
- `4.3` branch for Godot 4.3.x stable releases

### Q: Do I need to rebuild godot-cpp for every project?
**A:** No, you can build godot-cpp once and reuse the compiled libraries across multiple projects. Just link against the appropriate static libraries (.lib/.a files) in your project's build configuration.

## Build and Setup

### Q: Why am I getting "cannot find godot-cpp headers" errors?
**A:** This usually means your include paths aren't set correctly. Make sure:
- You've built godot-cpp successfully
- Your project's include directories point to `godot-cpp/include`
- You're linking against the correct godot-cpp library files

### Q: Should I use SCons or CMake for building?
**A:** SCons is the recommended build system as it's officially supported and tested. CMake support is available but may lag behind SCons in terms of features and platform support.

### Q: Can I use different compilers than what Godot was built with?
**A:** It's recommended to use the same compiler family (MSVC, GCC, Clang) that was used to build your Godot binary to avoid ABI compatibility issues, especially on Windows.

## Development

### Q: When should I inherit from Node vs Resource vs RefCounted?
**A:**
- **Node**: For objects that exist in the scene tree (visual elements, game logic, etc.)
- **Resource**: For data containers that can be saved/loaded (.tres/.res files)
- **RefCounted**: For utility objects with automatic memory management
- **Object**: Base class, rarely used directly

### Q: How do I expose custom classes to GDScript?
**A:** Use the `GDCLASS` macro in your class declaration and `GDREGISTER_CLASS` in your module initialization:

```cpp
// In header
class MyClass : public Node {
    GDCLASS(MyClass, Node)
    // class implementation
};

// In module initialization
GDREGISTER_CLASS(MyClass)
```

### Q: Why aren't my custom properties showing in the editor?
**A:** Properties need to be explicitly registered using `ClassDB::bind_method()` for getters/setters or `ADD_PROPERTY` macro:

```cpp
void MyClass::_bind_methods() {
    ClassDB::bind_method(D_METHOD("get_speed"), &MyClass::get_speed);
    ClassDB::bind_method(D_METHOD("set_speed", "speed"), &MyClass::set_speed);
    ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "speed"), "set_speed", "get_speed");
}
```

### Q: How do I handle signals in C++?
**A:** Define signals in `_bind_methods()` and emit them using `emit_signal()`:

```cpp
void MyClass::_bind_methods() {
    ADD_SIGNAL(MethodInfo("health_changed", PropertyInfo(Variant::INT, "new_health")));
}

void MyClass::take_damage(int amount) {
    health -= amount;
    emit_signal("health_changed", health);
}
```

## Performance

### Q: Is GDExtension slower than built-in GDScript?
**A:** No, GDExtension code typically runs significantly faster than GDScript since it's compiled native code. However, frequent calls across the engine boundary can add overhead.

### Q: How can I optimize performance?
**A:** Key optimization strategies:
- Minimize engine-extension boundary crossings
- Use batch operations when possible
- Cache frequently accessed engine objects
- Use appropriate data structures (PackedArrays for large datasets)
- Consider using direct memory access patterns where applicable

### Q: Should I move all logic to C++?
**A:** Not necessarily. Use C++ for performance-critical code and complex algorithms, but GDScript is fine for game logic, UI handling, and prototyping. The best approach often combines both.

## Memory Management

### Q: Do I need to manually delete Godot objects?
**A:** Usually no. Most Godot objects use reference counting (RefCounted) or are managed by the scene tree (Node). Manual deletion is typically only needed for raw pointers and custom memory allocations.

### Q: Why am I getting memory leaks?
**A:** Common causes:
- Not releasing references to RefCounted objects
- Circular references between objects
- Keeping references to deleted nodes
- Not properly cleaning up in destructors

### Q: How do I debug memory issues?
**A:** Use Godot's built-in memory profiler and tools like:
- Valgrind on Linux
- Application Verifier on Windows
- AddressSanitizer with GCC/Clang
- Memory leak detection tools provided by your IDE

## Platform Specific

### Q: Why won't my extension load on different platforms?
**A:** Common issues:
- Missing platform-specific libraries in your .gdextension file
- Architecture mismatches (x86 vs x64, ARM vs x86)
- Different runtime libraries (make sure all dependencies are included)
- Path separators (use forward slashes in .gdextension files)

### Q: How do I handle platform-specific code?
**A:** Use preprocessor definitions and conditional compilation:

```cpp
#ifdef WINDOWS_ENABLED
    // Windows-specific code
#elif defined(LINUX_ENABLED)
    // Linux-specific code
#elif defined(MACOS_ENABLED)
    // macOS-specific code
#endif
```

## Debugging

### Q: How do I debug my GDExtension code?
**A:** Debugging approaches:
- Use `print_line()` for simple debugging output
- Attach your IDE's debugger to the Godot process
- Use Godot's `--verbose` flag for more detailed logging
- Enable debug builds of both Godot and your extension

### Q: Why does my extension crash Godot?
**A:** Common causes:
- Null pointer dereferences
- Buffer overflows
- Use after free errors
- Stack overflows from infinite recursion
- ABI mismatches between extension and engine

### Q: How do I handle errors gracefully?
**A:** Use Godot's error macros and proper error checking:

```cpp
ERR_FAIL_NULL_V(pointer, default_value);
ERR_FAIL_COND_V(condition, default_value);
WARN_PRINT("Warning message");
```

## Advanced Topics

### Q: Can I create editor plugins with GDExtension?
**A:** Yes, but editor plugins are typically better implemented in GDScript due to their UI-heavy nature. Use GDExtension for the core functionality and GDScript for the editor integration.

### Q: How do I handle multithreading?
**A:** Be careful with threading in GDExtension:
- Most Godot API calls are not thread-safe
- Use `call_deferred()` to communicate with the main thread
- Protect shared data with mutexes
- Consider using Godot's threading utilities like WorkerThreadPool

### Q: Can I modify Godot classes from my extension?
**A:** You cannot modify existing Godot classes, but you can:
- Inherit from them and add functionality
- Use composition to extend functionality
- Create utility classes that work with Godot objects

## Troubleshooting

### Q: My changes aren't being reflected in Godot
**A:** Try these steps:
1. Rebuild your extension completely (`scons -c` then `scons`)
2. Close and reopen Godot
3. Check that your .gdextension file points to the correct library paths
4. Verify that your initialization function is being called

### Q: I get "symbol not found" errors
**A:** This usually indicates:
- Missing library dependencies
- ABI compatibility issues
- Incorrect linking configuration
- Platform-specific symbol visibility issues

### Q: How do I get help with specific problems?
**A:** Resources for help:
- Official Godot documentation and tutorials
- Godot Discord community (#gdextension channel)
- GitHub issues on the godot-cpp repository
- Stack Overflow with "godot" and "gdextension" tags
- Godot subreddit community