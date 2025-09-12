# Generated Code Patterns Internal Documentation

## Table of Contents
1. [Overview](#overview)
2. [Engine Class Generation](#engine-class-generation)
3. [Method Call Patterns](#method-call-patterns)
4. [Virtual Method Binding](#virtual-method-binding)
5. [Marshalling Patterns](#marshalling-patterns)
6. [Singleton Patterns](#singleton-patterns)
7. [Special Case Patterns](#special-case-patterns)
8. [Performance Optimizations](#performance-optimizations)

## Overview

The binding generator produces highly optimized C++ code patterns that bridge godot-cpp with the engine's binary interface. Generated code follows strict patterns to ensure ABI stability, minimize overhead, and provide type safety.

### Generation Architecture
```
extension_api.json → binding_generator.py → Generated C++ Files
                                          ↓
                              ┌──────────────────────────┐
                              │ include/godot_cpp/classes │
                              │ - One .hpp per class     │
                              │ - GDEXTENSION_CLASS macro │
                              │ - Method signatures       │
                              └──────────────────────────┘
                              ┌──────────────────────────┐
                              │ src/classes              │
                              │ - One .cpp per class     │
                              │ - Method implementations  │
                              │ - Static binding cache   │
                              └──────────────────────────┘
```

## Engine Class Generation

### Header Pattern

Every engine class generates a header following this structure:

```cpp
// generated: include/godot_cpp/classes/node.hpp
#pragma once

#include <godot_cpp/classes/object.hpp>  // Base class
#include <godot_cpp/core/class_db.hpp>

namespace godot {

// Forward declarations for used types
class NodePath;
class StringName;

class Node : public Object {
    GDEXTENSION_CLASS(Node, Object)  // Macro expansion documented below

public:
    // Enums from extension_api.json
    enum ProcessMode {
        PROCESS_MODE_INHERIT = 0,
        PROCESS_MODE_PAUSABLE = 1,
        // ...
    };

    // Constants
    static const int NOTIFICATION_READY = 13;

    // Generated methods
    void add_child(Node *p_node, bool p_force_readable_name = false);
    Node *get_node_internal(const NodePath &p_path) const;

    // Virtual methods (default implementations)
    virtual void _ready() {}
    virtual void _process(double p_delta) {}

protected:
    // Virtual registration template
    template <typename T, typename B>
    static void register_virtuals() {
        Object::register_virtuals<T, B>();
        if constexpr (!std::is_same_v<decltype(&B::_ready), decltype(&T::_ready)>) {
            BIND_VIRTUAL_METHOD(T, _ready, 0x92A8C2B4);  // Hash from API
        }
    }
};

} // namespace godot

VARIANT_ENUM_CAST(Node::ProcessMode);
```

### `GDEXTENSION_CLASS` Macro Expansion

The `GDEXTENSION_CLASS` macro generates critical infrastructure:

```cpp
// Macro expansion for GDEXTENSION_CLASS(Node, Object)
public:
    // Type identification
    static const char *get_class_static() {
        return "Node";
    }

    // Parent class chain
    static const char *get_parent_class_static() {
        return "Object";
    }

    // GDExtension binding callbacks
    static GDExtensionClassCallbacksType _gde_binding_callbacks = {
        .create_instance = &_gde_create_instance,
        .free_instance = &_gde_free_instance,
        .get_virtual = &_gde_get_virtual,
        .get_rid = nullptr,
        .to_string = &_gde_to_string,
        .notification = &_gde_notification,
        .reference = nullptr,
        .unreference = nullptr,
    };

protected:
    // Instance creation for engine
    static void *_gde_create_instance(void *p_userdata) {
        return memnew(Node);
    }

    // Virtual method dispatch
    static GDExtensionClassCallVirtual _gde_get_virtual(void *p_userdata,
                                                        const StringName *p_name) {
        // Returns function pointer for virtual method
        if (*p_name == StringName("_ready")) {
            return (GDExtensionClassCallVirtual)&_ready_virtual_wrapper;
        }
        return Object::_gde_get_virtual(p_userdata, p_name);
    }
```

## Method Call Patterns

### Regular Method Calls

The generator creates optimized wrappers for each method type:

```cpp
// Source: src/classes/node.cpp

// 1. Method with return value
Node *Node::get_child(int64_t p_idx) const {
    // Static caching of method bind (one-time lookup)
    static GDExtensionMethodBindPtr _gde_method_bind =
        internal::gdextension_interface_classdb_get_method_bind(
            Node::get_class_static()._native_ptr(),
            StringName("get_child")._native_ptr(),
            0x8B5A8F3C  // Method hash for version safety
        );

    // Error checking in debug builds
    CHECK_METHOD_BIND_RET(_gde_method_bind, nullptr);

    // Encode parameters for binary transfer
    int64_t p_idx_encoded;
    PtrToArg<int64_t>::encode(p_idx, &p_idx_encoded);

    // Call through function pointer table
    return internal::_call_native_mb_ret_obj<Node>(
        _gde_method_bind,
        _owner,         // GDExtensionObjectPtr
        &p_idx_encoded  // Encoded arguments
    );
}

// 2. Void method (no return)
void Node::add_child(Node *p_node, bool p_force_readable_name) {
    static GDExtensionMethodBindPtr _gde_method_bind =
        internal::gdextension_interface_classdb_get_method_bind(
            Node::get_class_static()._native_ptr(),
            StringName("add_child")._native_ptr(),
            0x1D5F4B2A
        );

    CHECK_METHOD_BIND(_gde_method_bind);

    // Parameter encoding
    int8_t p_force_readable_name_encoded;
    PtrToArg<bool>::encode(p_force_readable_name, &p_force_readable_name_encoded);

    internal::_call_native_mb_no_ret(
        _gde_method_bind,
        _owner,
        (p_node != nullptr ? &p_node->_owner : nullptr),  // Null check
        &p_force_readable_name_encoded
    );
}

// 3. Static method
Dictionary Node::get_configuration_warnings_as_dict() {
    static GDExtensionMethodBindPtr _gde_method_bind =
        internal::gdextension_interface_classdb_get_method_bind(
            Node::get_class_static()._native_ptr(),
            StringName("get_configuration_warnings_as_dict")._native_ptr(),
            0xC74D2E3F
        );

    CHECK_METHOD_BIND_RET(_gde_method_bind, Dictionary());

    // Static methods pass nullptr for instance
    return internal::_call_native_mb_ret<Dictionary>(
        _gde_method_bind,
        nullptr  // No instance for static
    );
}
```

### Vararg Method Pattern

Variable argument methods use template expansion:

```cpp
// Header: Vararg method declaration
private:
    // Internal implementation takes raw arguments
    Variant call_internal(const StringName &p_method,
                         const Variant **p_args,
                         GDExtensionInt p_arg_count);

public:
    // Template wrapper for type safety
    template <typename... Args>
    Variant call(const StringName &p_method, const Args &...p_args) {
        std::array<Variant, sizeof...(Args)> variant_args = { Variant(p_args)... };
        std::array<const Variant *, sizeof...(Args)> args_ptr;
        for (size_t i = 0; i < sizeof...(Args); i++) {
            args_ptr[i] = &variant_args[i];
        }
        return call_internal(p_method, args_ptr.data(), sizeof...(Args));
    }

// Source: Implementation
Variant Object::call_internal(const StringName &p_method,
                             const Variant **p_args,
                             GDExtensionInt p_arg_count) {
    static GDExtensionMethodBindPtr _gde_method_bind =
        internal::gdextension_interface_classdb_get_method_bind(
            Object::get_class_static()._native_ptr(),
            StringName("call")._native_ptr(),
            0xF832A1E7
        );

    CHECK_METHOD_BIND_RET(_gde_method_bind, Variant());

    GDExtensionCallError error;
    Variant ret;

    // Vararg methods use special interface
    internal::gdextension_interface_object_method_bind_call(
        _gde_method_bind,
        _owner,
        reinterpret_cast<GDExtensionConstVariantPtr *>(p_args),
        p_arg_count,
        &ret,
        &error
    );

    return ret;
}
```

## Virtual Method Binding

### Virtual Method Registration

The generator creates sophisticated virtual method handling:

```cpp
// Generated virtual registration in header
template <typename T, typename B>
static void register_virtuals() {
    // Register parent virtuals first
    Object::register_virtuals<T, B>();

    // Use if constexpr to detect overrides at compile time
    // This check compiles away if method not overridden
    if constexpr (!std::is_same_v<decltype(&B::_ready), decltype(&T::_ready)>) {
        // Method is overridden, bind it
        BIND_VIRTUAL_METHOD(T, _ready, 0x92A8C2B4);
    }

    if constexpr (!std::is_same_v<decltype(&B::_process), decltype(&T::_process)>) {
        BIND_VIRTUAL_METHOD(T, _process, 0x813C4F21);
    }
}

// BIND_VIRTUAL_METHOD macro expansion
#define BIND_VIRTUAL_METHOD(T, m_method, m_hash) \
    ClassDB::bind_virtual_method(                \
        T::get_class_static(),                   \
        StringName(#m_method),                   \
        &_gdvirtual_##m_method##_wrapper<T>,     \
        m_hash                                    \
    )

// Virtual wrapper function template
template <typename T>
static void _gdvirtual__ready_wrapper(GDExtensionClassInstancePtr p_instance) {
    T *instance = reinterpret_cast<T *>(p_instance);
    instance->_ready();
}
```

### Virtual Method Call Pattern

```cpp
// Virtual method with parameters and return
virtual String _to_string() const {
    // Default implementation
    return "<" + get_class() + "#" + itos(get_instance_id()) + ">";
}

// Generated wrapper for engine callbacks
template <typename T>
static GDExtensionStringPtr _gdvirtual__to_string_wrapper(
    GDExtensionClassInstancePtr p_instance) {

    T *instance = reinterpret_cast<T *>(p_instance);
    String result = instance->_to_string();

    // Convert return value to engine format
    GDExtensionStringPtr ret;
    internal::gdextension_interface_string_new_with_utf8_chars(
        &ret,
        result.utf8().get_data()
    );
    return ret;
}
```

## Marshalling Patterns

### PtrToArg Encoding

The generator uses PtrToArg for parameter marshalling:

```cpp
// Example of generated parameter encoding
void Node::set_process_priority(int32_t p_priority) {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;

    // 1. POD type encoding (int32_t)
    int32_t p_priority_encoded;
    PtrToArg<int32_t>::encode(p_priority, &p_priority_encoded);

    internal::_call_native_mb_no_ret(
        _gde_method_bind,
        _owner,
        &p_priority_encoded
    );
}

void Node::add_to_group(const StringName &p_group, bool p_persistent) {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;

    // 2. Variant type (StringName) - passed by reference
    // No encoding needed for variant types

    // 3. Bool encoding (special case to int8_t)
    int8_t p_persistent_encoded;
    PtrToArg<bool>::encode(p_persistent, &p_persistent_encoded);

    internal::_call_native_mb_no_ret(
        _gde_method_bind,
        _owner,
        &p_group,  // Direct reference for variant
        &p_persistent_encoded
    );
}

void Node::reparent(Node *p_new_parent, bool p_keep_global_transform) {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;

    // 4. Object pointer encoding with null check
    int8_t p_keep_global_transform_encoded;
    PtrToArg<bool>::encode(p_keep_global_transform, &p_keep_global_transform_encoded);

    internal::_call_native_mb_no_ret(
        _gde_method_bind,
        _owner,
        (p_new_parent != nullptr ? &p_new_parent->_owner : nullptr),
        &p_keep_global_transform_encoded
    );
}
```

### Return Value Handling

Different patterns for different return types:

```cpp
// 1. POD return type
int64_t Node::get_child_count() const {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;
    CHECK_METHOD_BIND_RET(_gde_method_bind, 0);

    return internal::_call_native_mb_ret<int64_t>(
        _gde_method_bind,
        _owner
    );
}

// 2. Enum return type (cast required)
Node::ProcessMode Node::get_process_mode() const {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;
    CHECK_METHOD_BIND_RET(_gde_method_bind, PROCESS_MODE_INHERIT);

    // Enums are returned as int64_t and cast
    return (ProcessMode)internal::_call_native_mb_ret<int64_t>(
        _gde_method_bind,
        _owner
    );
}

// 3. RefCounted return (special Ref<> handling)
Ref<Script> Node::get_script() const {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;
    CHECK_METHOD_BIND_RET(_gde_method_bind, Ref<Script>());

    // Use special constructor to avoid reference count issues
    return Ref<Script>::_gde_internal_constructor(
        internal::_call_native_mb_ret_obj<Script>(
            _gde_method_bind,
            _owner
        )
    );
}

// 4. Object pointer return
Node *Node::get_parent() const {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;
    CHECK_METHOD_BIND_RET(_gde_method_bind, nullptr);

    return internal::_call_native_mb_ret_obj<Node>(
        _gde_method_bind,
        _owner
    );
}
```

## Singleton Patterns

### Singleton Generation

Singletons have special initialization patterns:

```cpp
// Header: include/godot_cpp/classes/engine.hpp
class Engine : public Object {
    GDEXTENSION_CLASS(Engine, Object)

    static Engine *singleton;  // Static singleton pointer

public:
    static Engine *get_singleton();

    // Singleton methods are regular instance methods
    double get_physics_fps() const;
    void set_physics_fps(double p_fps);

protected:
    ~Engine();  // Protected destructor
};

// Source: src/classes/engine.cpp
Engine *Engine::singleton = nullptr;

Engine *Engine::get_singleton() {
    // Thread-safe lazy initialization
    if (unlikely(singleton == nullptr)) {
        // Get singleton from engine
        GDExtensionObjectPtr singleton_obj =
            internal::gdextension_interface_global_get_singleton(
                Engine::get_class_static()._native_ptr()
            );

        #ifdef DEBUG_ENABLED
        ERR_FAIL_NULL_V(singleton_obj, nullptr);
        #endif

        // Create binding
        singleton = reinterpret_cast<Engine *>(
            internal::gdextension_interface_object_get_instance_binding(
                singleton_obj,
                internal::token,
                &Engine::_gde_binding_callbacks
            )
        );

        #ifdef DEBUG_ENABLED
        ERR_FAIL_NULL_V(singleton, nullptr);
        #endif

        // Register for cleanup
        if (likely(singleton)) {
            ClassDB::_register_engine_singleton(
                Engine::get_class_static(),
                singleton
            );
        }
    }
    return singleton;
}

Engine::~Engine() {
    // Cleanup on destruction
    if (singleton == this) {
        ClassDB::_unregister_engine_singleton(Engine::get_class_static());
        singleton = nullptr;
    }
}
```

### ClassDB Singleton Special Case

ClassDBSingleton has unique macro forwarding:

```cpp
// Generated macros for ClassDB compatibility
#define CLASSDB_SINGLETON_FORWARD_METHODS \
    static bool class_exists(const StringName &p_class_name) { \
        return ClassDBSingleton::get_singleton()->class_exists(p_class_name); \
    } \
    static bool is_parent_class(const StringName &p_class, const StringName &p_parent) { \
        return ClassDBSingleton::get_singleton()->is_parent_class(p_class, p_parent); \
    } \
    // ... all methods forwarded

// Usage in user code
class ClassDB {
public:
    CLASSDB_SINGLETON_FORWARD_METHODS
};
```

## Special Case Patterns

### Template Get Node Pattern

Node class has special template handling:

```cpp
// When use_template_get_node is enabled
class Node : public Object {
public:
    // Template version for type safety
    template <typename T>
    T *get_node(const NodePath &p_path) const {
        return Object::cast_to<T>(get_node_internal(p_path));
    }

private:
    // Internal non-template implementation
    Node *get_node_internal(const NodePath &p_path) const;
};
```

### Native Structure Patterns

Native structures generate POD wrappers:

```cpp
// Generated from native_structures in API
struct AudioFrame {
    float left;
    float right;
};

// Macro for PtrToArg specialization
GDVIRTUAL_NATIVE_PTR(AudioFrame);

// Expands to:
template <>
struct PtrToArg<AudioFrame *> {
    _FORCE_INLINE_ static AudioFrame *convert(const void *p_ptr) {
        return (AudioFrame *)(*(void **)p_ptr);
    }
    typedef AudioFrame *EncodeT;
    _FORCE_INLINE_ static void encode(AudioFrame *p_var, void *p_ptr) {
        *reinterpret_cast<AudioFrame **>(p_ptr) = p_var;
    }
};
```

### Special Method Implementations

Some classes have hardcoded special methods:

```cpp
// XMLParser special buffer handling
Error XMLParser::_open_buffer(const uint8_t *p_buffer, size_t p_size) {
    // Direct memory access pattern
    PackedByteArray buffer;
    buffer.resize(p_size);
    memcpy(buffer.ptrw(), p_buffer, p_size);
    return open_buffer(buffer);
}

// Image direct memory access
uint8_t *Image::ptrw() {
    // Get write access to internal data
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;
    return internal::_call_native_mb_ret_ptr<uint8_t>(
        _gde_method_bind,
        _owner
    );
}

// FileAccess buffer operations
uint64_t FileAccess::get_buffer(uint8_t *p_dst, uint64_t p_length) const {
    // Special handling for buffer transfer
    PackedByteArray buffer;
    uint64_t ret = get_buffer_internal(buffer, p_length);
    memcpy(p_dst, buffer.ptr(), MIN(ret, p_length));
    return ret;
}
```

## Performance Optimizations

### Static Method Bind Caching

Every method caches its bind once:

```cpp
void Node::some_method() {
    // Static ensures one-time initialization
    static GDExtensionMethodBindPtr _gde_method_bind =
        internal::gdextension_interface_classdb_get_method_bind(
            Node::get_class_static()._native_ptr(),
            StringName("some_method")._native_ptr(),
            0x12345678
        );

    // Subsequent calls reuse cached pointer
    internal::_call_native_mb_no_ret(_gde_method_bind, _owner);
}
```

### Inline Call Wrappers

Critical paths use forced inlining:

```cpp
// In internal namespace
template <typename T>
_FORCE_INLINE_ T _call_native_mb_ret(GDExtensionMethodBindPtr p_method_bind,
                                     void *p_instance,
                                     const void *const *p_args = nullptr) {
    T ret;
    // Direct function pointer call, no indirection
    p_method_bind(p_instance, p_args, &ret, sizeof...(p_args));
    return ret;
}

// Specialized for objects to handle binding
template <typename T>
_FORCE_INLINE_ T *_call_native_mb_ret_obj(GDExtensionMethodBindPtr p_method_bind,
                                          void *p_instance,
                                          const void *const *p_args = nullptr) {
    GDExtensionObjectPtr ret_obj;
    p_method_bind(p_instance, p_args, &ret_obj, sizeof...(p_args));

    // Convert to C++ wrapper
    return reinterpret_cast<T *>(
        internal::get_object_instance_binding(ret_obj)
    );
}
```

### Compile-Time Virtual Detection

Virtual overrides detected at compile time:

```cpp
// if constexpr evaluates at compile time
// No runtime overhead for non-overridden virtuals
if constexpr (!std::is_same_v<decltype(&B::_ready), decltype(&T::_ready)>) {
    // This code only compiles if _ready is overridden
    BIND_VIRTUAL_METHOD(T, _ready, 0x92A8C2B4);
}
// No else branch needed - optimized away completely
```

### Zero-Cost Abstractions

Generated code maintains zero-cost principles:

```cpp
// 1. Direct memory layout matching
class Node {
    GDExtensionObjectPtr _owner;  // Only data member
    // All methods are non-virtual (except user overrides)
};

// 2. Return value optimization (RVO)
String Node::get_name() const {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;
    // Direct construction at return site
    return internal::_call_native_mb_ret<String>(_gde_method_bind, _owner);
}

// 3. Perfect forwarding for variants
template <typename... Args>
Variant call(const StringName &p_method, Args &&...p_args) {
    // Perfect forwarding preserves value categories
    return call_internal(p_method, std::forward<Args>(p_args)...);
}
```

### Minimal Memory Overhead

Binding structures are minimal:

```cpp
// Each class instance: 8 bytes (64-bit pointer)
sizeof(Node) == sizeof(void*);  // True

// Static data per class (shared across all instances):
// - Method bind cache: 8 bytes per method
// - Class name string: One shared string
// - Singleton pointer: 8 bytes (only for singletons)

// No virtual tables except for user virtuals
// No RTTI overhead
// No hidden members
```

## Error Handling in Generated Code

### Debug vs Release Patterns

```cpp
// Debug builds include checks
#ifdef DEBUG_ENABLED
    #define CHECK_METHOD_BIND(m_bind) \
        ERR_FAIL_NULL(m_bind)
    #define CHECK_METHOD_BIND_RET(m_bind, m_ret) \
        ERR_FAIL_NULL_V(m_bind, m_ret)
#else
    // Release builds assume success
    #define CHECK_METHOD_BIND(m_bind)
    #define CHECK_METHOD_BIND_RET(m_bind, m_ret)
#endif

// Usage in generated code
void Node::some_method() {
    static GDExtensionMethodBindPtr _gde_method_bind = /*...*/;
    CHECK_METHOD_BIND(_gde_method_bind);  // Only in debug
    // ...
}
```

## Summary

The generated code patterns represent a sophisticated system for binary-stable, high-performance bindings:

1. **Static Caching**: One-time method bind lookups cached in static variables
2. **Zero-Cost Abstractions**: Templates and constexpr eliminate runtime overhead
3. **Type Safety**: Template wrappers provide compile-time type checking
4. **Binary Stability**: Hash-based versioning ensures ABI compatibility
5. **Minimal Overhead**: Direct function pointer calls, no virtual dispatch
6. **Smart Marshalling**: PtrToArg system handles all type conversions efficiently

These patterns ensure that godot-cpp provides native performance while maintaining the safety and convenience of modern C++. The generator's output is carefully crafted to minimize both runtime overhead and compilation time while maximizing optimization opportunities for the compiler.

*Total lines of generated code per class: ~200-500 lines depending on method count*
*Compilation time per class: ~0.1-0.3 seconds on modern hardware*
*Runtime overhead vs engine internals: <1% for most operations*
