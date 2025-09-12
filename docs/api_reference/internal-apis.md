# Internal APIs Reference

## Table of Contents
1. [GDExtension Interface Functions](#gdextension-interface-functions)
2. [Internal Namespace APIs](#internal-namespace-apis)
3. [Memory Management APIs](#memory-management-apis)
4. [Type System APIs](#type-system-apis)
5. [Method Binding APIs](#method-binding-apis)
6. [Object Management APIs](#object-management-apis)
7. [String and Name APIs](#string-and-name-apis)
8. [Variant APIs](#variant-apis)

## Overview

This reference documents the 160+ internal API functions that form the bridge between godot-cpp and the Godot engine. These functions handle all cross-binary communication, memory management, and type system operations.

## GDExtension Interface Functions

Core interface functions provided by the engine:

> **Interface Stability**: These C-compatible functions form the stable ABI between extensions and the engine. Function signatures never change within a major version, ensuring binary compatibility across minor updates.

### Initialization Functions

| Function | Purpose | When Called |
|----------|---------|-------------|
| `get_proc_address` | Resolve other interface functions | Extension load |
| `get_library_path` | Get extension file path | Debugging/resources |
| `get_godot_version` | Version compatibility check | Initialization |

```cpp
// Get interface function by name
GDExtensionInterfaceFunctionPtr gdextension_interface_get_proc_address(
    const char *p_function_name
);

// Get library path
void gdextension_interface_get_library_path(
    GDExtensionClassLibraryPtr p_library,
    GDExtensionStringPtr r_path
);

// Get Godot version
void gdextension_interface_get_godot_version(
    GDExtensionGodotVersion *r_godot_version
);
```

### Memory Management

> **Memory Strategy**: All memory allocated through these functions is tracked by the engine's memory statistics and respects engine-wide limits. Always use these instead of malloc/free to ensure proper cleanup on engine shutdown.

```cpp
// Memory allocation - integrates with engine's memory pools
void *gdextension_interface_mem_alloc(size_t p_size);
void *gdextension_interface_mem_realloc(void *p_ptr, size_t p_size);
void gdextension_interface_mem_free(void *p_ptr);

// Reference counting
void gdextension_interface_ref_set_object(
    GDExtensionRefPtr p_ref,
    GDExtensionObjectPtr p_object
);
GDExtensionObjectPtr gdextension_interface_ref_get_object(
    GDExtensionConstRefPtr p_ref
);
```

### Class Registration

```cpp
// Register extension class
void gdextension_interface_classdb_register_extension_class(
    GDExtensionClassLibraryPtr p_library,
    const GDExtensionStringNamePtr p_class_name,
    const GDExtensionStringNamePtr p_parent_class_name,
    const GDExtensionClassCreationInfo *p_extension_funcs
);

// Register method
void gdextension_interface_classdb_register_extension_class_method(
    GDExtensionClassLibraryPtr p_library,
    const GDExtensionStringNamePtr p_class_name,
    const GDExtensionClassMethodInfo *p_method_info
);

// Register property
void gdextension_interface_classdb_register_extension_class_property(
    GDExtensionClassLibraryPtr p_library,
    const GDExtensionStringNamePtr p_class_name,
    const GDExtensionPropertyInfo *p_info,
    const GDExtensionStringNamePtr p_setter,
    const GDExtensionStringNamePtr p_getter
);

// Register signal
void gdextension_interface_classdb_register_extension_class_signal(
    GDExtensionClassLibraryPtr p_library,
    const GDExtensionStringNamePtr p_class_name,
    const GDExtensionStringNamePtr p_signal_name,
    const GDExtensionPropertyInfo *p_argument_info,
    GDExtensionInt p_argument_count
);
```

## Internal Namespace APIs

The `internal` namespace contains wrapper functions:

> **Internal API Purpose**: These wrappers provide type-safe C++ interfaces over the raw C function pointers. They handle casting, error checking, and provide a cleaner API for the rest of godot-cpp to use.

### Core Initialization

```cpp
namespace internal {
    // Library handle
    extern GDExtensionClassLibraryPtr library;

    // Token for instance binding
    extern void *token;

    // Interface function table
    extern GDExtensionInterface interface;

    // Initialize interface
    void initialize_interface(
        GDExtensionInterfaceGetProcAddress p_get_proc_address
    );
}
```

### Method Call Wrappers

> **Performance Optimization**: These template wrappers are force-inlined to eliminate function call overhead. The compiler optimizes them to direct calls through the function pointer, achieving near-native performance for engine method calls.

```cpp
// Call native method with return value - zero overhead wrapper
template <typename T>
T _call_native_mb_ret(
    GDExtensionMethodBindPtr p_method_bind,
    void *p_instance,
    const void *const *p_args = nullptr
) {
    T ret;
    p_method_bind(p_instance, p_args, &ret, sizeof...(p_args));
    return ret;
}

// Call native method without return
void _call_native_mb_no_ret(
    GDExtensionMethodBindPtr p_method_bind,
    void *p_instance,
    const void *const *p_args = nullptr
) {
    p_method_bind(p_instance, p_args, nullptr, sizeof...(p_args));
}

// Call method returning object
template <typename T>
T *_call_native_mb_ret_obj(
    GDExtensionMethodBindPtr p_method_bind,
    void *p_instance,
    const void *const *p_args = nullptr
) {
    GDExtensionObjectPtr ret;
    p_method_bind(p_instance, p_args, &ret, sizeof...(p_args));
    return (T *)get_object_instance_binding(ret);
}
```

### Utility Function Calls

```cpp
// Call utility function with return
template <typename T>
T _call_utility_ret(
    GDExtensionPtrUtilityFunction p_function,
    const void *const *p_args = nullptr
) {
    T ret;
    p_function(&ret, p_args, sizeof...(p_args));
    return ret;
}

// Call utility function without return
void _call_utility_no_ret(
    GDExtensionPtrUtilityFunction p_function,
    const void *const *p_args = nullptr
) {
    Variant ret;
    p_function(&ret, p_args, sizeof...(p_args));
}

// Call utility returning object
Object *_call_utility_ret_obj(
    GDExtensionPtrUtilityFunction p_function,
    const void *const *p_args = nullptr
) {
    GDExtensionObjectPtr ret;
    p_function(&ret, p_args, sizeof...(p_args));
    return (Object *)get_object_instance_binding(ret);
}
```

## Memory Management APIs

### Allocation Functions

```cpp
class Memory {
public:
    // Static memory allocation
    static void *alloc_static(size_t p_bytes, bool p_pad_align = false);
    static void *realloc_static(void *p_memory, size_t p_bytes, bool p_pad_align = false);
    static void free_static(void *p_ptr, bool p_pad_align = false);

    // Get allocation size
    static uint64_t get_mem_available();
    static uint64_t get_mem_usage();
    static uint64_t get_mem_max_usage();
};

// Placement new/delete
template <typename T>
void *memnew_placement(void *p_ptr, T *p_class) {
    return new (p_ptr) T;
}

template <typename T>
T *memnew() {
    return memnew_placement(Memory::alloc_static(sizeof(T)), T);
}

template <typename T>
void memdelete(T *p_class) {
    if (p_class) {
        p_class->~T();
        Memory::free_static(p_class);
    }
}
```

### Reference Counting

```cpp
class SafeRefCount {
    std::atomic<uint32_t> count;

public:
    bool ref();          // Increment
    uint32_t refval();   // Get current value
    bool unref();        // Decrement, return true if zero
    uint32_t unrefval(); // Decrement, return new value
    bool init();         // Initialize to 1
};

class SafeNumeric<T> {
    std::atomic<T> value;

public:
    void set(T p_value);
    T get() const;
    T increment();
    T decrement();
    T add(T p_value);
    T sub(T p_value);
    T exchange(T p_value);
};
```

## Type System APIs

### Type Information

```cpp
template <typename T>
struct GetTypeInfo {
    static constexpr GDExtensionVariantType VARIANT_TYPE = /*...*/;
    static constexpr GDExtensionClassMethodArgumentMetadata METADATA = /*...*/;

    static inline PropertyInfo get_class_info() {
        return PropertyInfo(VARIANT_TYPE, "", PROPERTY_HINT_NONE,
                          "", PROPERTY_USAGE_DEFAULT,
                          T::get_class_static());
    }
};

// Type traits
template <typename T>
struct TypeInherits {
    static bool check(const T *p_ptr);
};

template <typename T>
struct TypesAreSame {
    static constexpr bool value = false;
};
```

### Property Information

```cpp
struct PropertyInfo {
    Variant::Type type = Variant::NIL;
    String name;
    StringName class_name;
    PropertyHint hint = PROPERTY_HINT_NONE;
    String hint_string;
    uint32_t usage = PROPERTY_USAGE_DEFAULT;

    PropertyInfo() = default;
    PropertyInfo(Variant::Type p_type, const String &p_name,
                PropertyHint p_hint = PROPERTY_HINT_NONE,
                const String &p_hint_string = "",
                uint32_t p_usage = PROPERTY_USAGE_DEFAULT,
                const StringName &p_class_name = StringName());
};

struct MethodInfo {
    String name;
    PropertyInfo return_val;
    uint32_t flags = METHOD_FLAGS_DEFAULT;
    int id = 0;
    List<PropertyInfo> arguments;
    Vector<Variant> default_arguments;
};
```

## Method Binding APIs

### MethodBind Creation

```cpp
template <typename T, typename R, typename... Args>
MethodBind *create_method_bind(R (T::*p_method)(Args...)) {
    MethodBind *bind = memnew(MethodBindT<T, R, Args...>(p_method));
    return bind;
}

template <typename T, typename R, typename... Args>
MethodBind *create_static_method_bind(R (*p_method)(Args...)) {
    MethodBind *bind = memnew(MethodBindTS<R, Args...>(p_method));
    return bind;
}

template <typename T>
MethodBind *create_vararg_method_bind(
    Variant (T::*p_method)(const Variant **, GDExtensionInt, GDExtensionCallError &),
    const MethodInfo &p_info,
    bool p_return_nil_is_variant
) {
    MethodBind *bind = memnew(MethodBindVarArg<T>(p_method, p_info, p_return_nil_is_variant));
    return bind;
}
```

### Method Registration

```cpp
class ClassDB {
public:
    template <typename N, typename M, typename... VarArgs>
    static MethodBind *bind_method(N p_method_name, M p_method, VarArgs... p_args);

    template <typename N, typename M, typename... VarArgs>
    static MethodBind *bind_static_method(StringName p_class, N p_method_name,
                                          M p_method, VarArgs... p_args);

    template <typename M>
    static MethodBind *bind_vararg_method(uint32_t p_flags, StringName p_name,
                                          M p_method, const MethodInfo &p_info,
                                          const std::vector<Variant> &p_default_args,
                                          bool p_return_nil_is_variant = true);
};
```

## Object Management APIs

### Instance Binding

```cpp
// Get instance binding
void *get_object_instance_binding(
    GDExtensionObjectPtr p_object
) {
    return internal::gdextension_interface_object_get_instance_binding(
        p_object, internal::token, nullptr
    );
}

// Set instance binding
void set_object_instance_binding(
    GDExtensionObjectPtr p_object,
    void *p_binding,
    const GDExtensionInstanceBindingCallbacks *p_callbacks
) {
    internal::gdextension_interface_object_set_instance_binding(
        p_object, internal::token, p_binding, p_callbacks
    );
}

// Get instance ID
uint64_t get_object_instance_id(GDExtensionObjectPtr p_object) {
    return internal::gdextension_interface_object_get_instance_id(p_object);
}

// Cast object
GDExtensionObjectPtr object_cast_to(
    GDExtensionObjectPtr p_object,
    void *p_class_tag
) {
    return internal::gdextension_interface_object_cast_to(
        p_object, p_class_tag
    );
}
```

### Object Creation

```cpp
// Create object
Object *create_object(const StringName &p_class_name) {
    GDExtensionObjectPtr obj =
        internal::gdextension_interface_classdb_construct_object(
            p_class_name._native_ptr()
        );
    return (Object *)get_object_instance_binding(obj);
}

// Destroy object
void destroy_object(Object *p_object) {
    internal::gdextension_interface_object_destroy(p_object->_owner);
}

// Get singleton
Object *get_singleton(const StringName &p_name) {
    GDExtensionObjectPtr singleton =
        internal::gdextension_interface_global_get_singleton(
            p_name._native_ptr()
        );
    return (Object *)get_object_instance_binding(singleton);
}
```

## String and Name APIs

### String Operations

```cpp
// String creation
void string_new_with_utf8_chars(String *r_dest, const char *p_contents);
void string_new_with_utf16_chars(String *r_dest, const char16_t *p_contents);
void string_new_with_utf32_chars(String *r_dest, const char32_t *p_contents);
void string_new_with_wide_chars(String *r_dest, const wchar_t *p_contents);

// String conversion
void string_to_utf8_chars(const String *p_self, char *r_text, int64_t p_max_write_length);
void string_to_utf16_chars(const String *p_self, char16_t *r_text, int64_t p_max_write_length);
void string_to_utf32_chars(const String *p_self, char32_t *r_text, int64_t p_max_write_length);
void string_to_wide_chars(const String *p_self, wchar_t *r_text, int64_t p_max_write_length);
```

### StringName Operations

```cpp
// StringName creation
void string_name_new_with_utf8_chars(StringName *r_dest, const char *p_contents);

// Get native pointer
GDExtensionStringNamePtr StringName::_native_ptr() const;

// Comparison
bool operator==(const StringName &p_a, const StringName &p_b);
bool operator!=(const StringName &p_a, const StringName &p_b);
bool operator<(const StringName &p_a, const StringName &p_b);
```

## Variant APIs

### Variant Construction

```cpp
// Construct from type
void variant_new_nil(Variant *r_dest);
void variant_new_bool(Variant *r_dest, bool p_value);
void variant_new_int(Variant *r_dest, int64_t p_value);
void variant_new_float(Variant *r_dest, double p_value);
void variant_new_string(Variant *r_dest, const String *p_value);
void variant_new_object(Variant *r_dest, Object *p_value);

// Copy construction
void variant_new_copy(Variant *r_dest, const Variant *p_src);
```

### Variant Operations

```cpp
// Type checking
Variant::Type variant_get_type(const Variant *p_self);
bool variant_can_convert(Variant::Type p_from, Variant::Type p_to);
bool variant_can_convert_strict(Variant::Type p_from, Variant::Type p_to);

// Conversion
void variant_convert(const Variant *p_from, Variant *r_to, Variant::Type p_type);

// Comparison
bool variant_equals(const Variant *p_a, const Variant *p_b);
bool variant_less(const Variant *p_a, const Variant *p_b);
uint32_t variant_hash(const Variant *p_self);

// String conversion
void variant_stringify(const Variant *p_self, String *r_string);
```

### Variant Calls

```cpp
// Call method
void variant_call(
    Variant *p_self,
    const StringName *p_method,
    const Variant **p_args,
    int64_t p_argument_count,
    Variant *r_return,
    GDExtensionCallError *r_error
);

// Call static method
void variant_call_static(
    Variant::Type p_type,
    const StringName *p_method,
    const Variant **p_args,
    int64_t p_argument_count,
    Variant *r_return,
    GDExtensionCallError *r_error
);

// Get member
void variant_get(
    const Variant *p_self,
    const Variant *p_key,
    Variant *r_ret,
    bool *r_valid
);

// Set member
void variant_set(
    Variant *p_self,
    const Variant *p_key,
    const Variant *p_value,
    bool *r_valid
);
```

## Summary

The internal APIs provide:
- **160+ interface functions** from GDExtension
- **Template wrappers** for type-safe calls
- **Memory management** with custom allocators
- **Type system** with compile-time introspection
- **Method binding** for dynamic dispatch
- **Object lifecycle** management
- **String handling** with UTF-8/16/32 support
- **Variant system** for dynamic typing

These APIs form the foundation for all godot-cpp functionality, providing the bridge between C++ code and the Godot engine.
