# Binding Generator

## Overview

The binding generator is a sophisticated Python-based code generation system that transforms Godot's API definition into type-safe C++ bindings. It processes a massive JSON API specification (337,509+ lines) and generates optimized C++ code with complete type information, method bindings, and proper memory layouts.

**Key Source Files:**
- `binding_generator.py` - Main generator implementation (3018 lines)
- `gdextension/extension_api.json` - Complete API definition (337,509 lines)
- `build_profile.py` - Build profile support (139 lines)
- `doc_source_generator.py` - Documentation generation (54 lines)

## Extension API JSON Schema

### Schema Structure

The `extension_api.json` file contains the complete Godot API definition with these major sections:

#### Header Section (Lines 2-10)

```json
{
  "header": {
    "version_major": 4,
    "version_minor": 5,
    "version_patch": 0,
    "version_status": "rc2",
    "version_build": "official",
    "version_full_name": "Godot Engine v4.5.rc2.official",
    "precision": "single"  // or "double"
  }
}
```

#### Builtin Class Sizes (Lines 11-672)

Platform and configuration-specific memory sizes:

```json
"builtin_class_sizes": [
  {
    "build_configuration": "float_32",
    "sizes": [
      {"name": "Variant", "size": 24},
      {"name": "String", "size": 8},
      {"name": "Vector2", "size": 8},
      {"name": "Vector3", "size": 12},
      {"name": "Transform3D", "size": 48},
      // ... all builtin types
    ]
  }
]
```

#### Member Offsets (Lines 673-1935)

Exact memory layout for struct members:

```json
"builtin_class_member_offsets": [
  {
    "build_configuration": "float_32",
    "classes": [
      {
        "name": "Vector2",
        "members": [
          {"member": "x", "offset": 0, "meta": "float"},
          {"member": "y", "offset": 4, "meta": "float"}
        ]
      }
    ]
  }
]
```

#### Global Enums (Lines 1936-4117)

All global constants and enumerations:

```json
"global_enums": [
  {
    "name": "Side",
    "is_bitfield": false,
    "values": [
      {"name": "SIDE_LEFT", "value": 0},
      {"name": "SIDE_TOP", "value": 1},
      {"name": "SIDE_RIGHT", "value": 2},
      {"name": "SIDE_BOTTOM", "value": 3}
    ]
  }
]
```

#### Utility Functions (Lines 4118-5898)

Global utility functions:

```json
"utility_functions": [
  {
    "name": "abs",
    "return_type": "Variant",
    "category": "math",
    "is_vararg": false,
    "hash": 3724354162,
    "arguments": [
      {"name": "x", "type": "Variant"}
    ]
  }
]
```

#### Builtin Classes (Lines 5899-25786)

Complete variant type definitions:

```json
"builtin_classes": [
  {
    "name": "String",
    "indexing_return_type": "String",
    "is_keyed": false,
    "operators": [...],
    "constructors": [...],
    "methods": [...],
    "members": [],
    "constants": []
  }
]
```

#### Engine Classes (Lines 25787-337509)

All Godot engine classes:

```json
"classes": [
  {
    "name": "Node",
    "is_refcounted": false,
    "is_instantiable": true,
    "inherits": "Object",
    "api_type": "core",
    "enums": [...],
    "methods": [...],
    "properties": [...],
    "signals": [...]
  }
]
```

## Code Generation Pipeline

### Main Generation Flow (`binding_generator.py:280-307`)

```python
def generate_bindings(api_filepath, use_template_get_node, bits="64",
                     precision="single", output_dir="."):
    api = None

    # Load and validate API
    with open(api_filepath, encoding="utf-8") as api_file:
        api = json.load(api_file)

    # Check precision compatibility
    if api["header"]["precision"] != precision:
        print("Error: Precision mismatch")
        exit(1)

    # Generate all binding components
    generate_global_constants(api, target_dir)          # Global enums/constants
    generate_version_header(api, target_dir)            # Version information
    generate_global_constant_binds(api, target_dir)     # Constant bindings
    generate_builtin_bindings(api, target_dir, real_t + "_" + bits)
    generate_engine_classes_bindings(api, target_dir, use_template_get_node)
    generate_utility_functions(api, target_dir)         # Utility function bindings
```

### Generation Phases

#### Phase 1: Global Constants (`binding_generator.py:3018`)

Generates `global_constants.hpp` with all enums and constants.

#### Phase 2: Version Header

Creates version information matching the engine.

#### Phase 3: Builtin Bindings (`binding_generator.py:507-1292`)

Generates headers and implementations for all variant types:
- Constructors with exact signatures
- Methods with proper return types
- Operators with type safety
- Memory layout specifications

#### Phase 4: Engine Class Bindings (`binding_generator.py:1336-2242`)

Generates bindings for all engine classes:
- Inheritance hierarchies
- Virtual method support
- Property accessors
- Signal definitions

#### Phase 5: Utility Functions (`binding_generator.py:2243-2373`)

Creates bindings for global utility functions.

## Template Generation Patterns

### Builtin Class Template (`binding_generator.py:589-994`)

```cpp
class ${ClassName} {
    static constexpr size_t ${CONSTANT_NAME}_SIZE = ${size};
    uint8_t opaque[${CONSTANT_NAME}_SIZE] = {};

    friend class Variant;

    // Static method bindings
    static struct _MethodBindings {
        GDExtensionTypeFromVariantConstructorFunc from_variant_constructor;
        GDExtensionVariantFromTypeConstructorFunc to_variant_constructor;
        GDExtensionPtrConstructor constructor_${n};
        GDExtensionPtrBuiltInMethod method_${name};
        GDExtensionPtrOperatorEvaluator operator_${op}_${type};
    } _method_bindings;

    static void init_bindings();

public:
    // Constructors
    _FORCE_INLINE_ ${ClassName}() {
        _method_bindings.constructor_0(_native_ptr());
    }

    _FORCE_INLINE_ ${ClassName}(${args}) {
        _method_bindings.constructor_${n}(_native_ptr(), ${encoded_args});
    }

    // Methods
    _FORCE_INLINE_ ${return_type} ${method_name}(${args}) const {
        ${return_type} ret;
        _method_bindings.method_${name}(
            (GDExtensionTypePtr)&opaque,
            ${encoded_args},
            &ret
        );
        return ret;
    }

    // Operators
    _FORCE_INLINE_ ${ClassName} operator${op}(${arg_type} p_other) const {
        ${ClassName} ret;
        _method_bindings.operator_${op}_${arg_type}(
            _native_ptr(),
            p_other._native_ptr(),
            ret._native_ptr()
        );
        return ret;
    }
};
```

### Engine Class Template (`binding_generator.py:1566-2061`)

```cpp
class ${ClassName} : public ${ParentClass} {
    GDCLASS(${ClassName}, ${ParentClass})

public:
    // Enums
    enum ${EnumName} {
        ${ENUM_VALUE} = ${value},
    };

    // Methods
    ${return_type} ${method_name}(${args}) const;

    // Virtual methods
    GDVIRTUAL${arg_count}R(${return_type}, _${vmethod_name}, ${arg_types})

    // Properties
    void set_${property}(${type} p_value);
    ${type} get_${property}() const;

    // Signals
    // Signal: ${signal_name}(${args})

protected:
    static void _bind_methods();

private:
    // Method bindings
    static struct _MethodBindings {
        MethodBind *method_${name};
    } _method_bindings;
};
```

## Special Case Handling

### Singleton Classes (`binding_generator.py:1336-1341`)

```python
# ClassDB is renamed to ClassDBSingleton to avoid conflicts
CLASS_ALIASES = {
    "ClassDB": "ClassDBSingleton",
}

for singleton in api["singletons"]:
    if singleton["name"] in CLASS_ALIASES:
        singleton["alias_for"] = singleton["name"]
        singleton["name"] = CLASS_ALIASES[singleton["name"]]
```

### POD Types (`binding_generator.py:2558-2578`)

```python
def is_pod_type(type_name):
    """Identifies plain-old-data types for optimized handling"""
    return type_name in [
        "Nil", "void", "bool", "real_t", "float", "double",
        "int", "int8_t", "uint8_t", "int16_t", "uint16_t",
        "int32_t", "int64_t", "uint32_t", "uint64_t"
    ]
```

### Enum Handling (`binding_generator.py:2635-2666`)

```python
def get_enum_fullname(enum_name: str):
    """Converts enum notation to C++ type"""
    if is_bitfield(enum_name):
        # "bitfield::Flags" → "BitField<Flags>"
        return enum_name.replace("bitfield::", "BitField<") + ">"
    else:
        # "enum::Type" → "Type"
        return enum_name.replace("enum::", "")
```

### String Type Special Handling (`binding_generator.py:522-526`)

```python
if class_name == "String":
    # String requires additional character utilities
    result.append("#include <godot_cpp/classes/global_constants.hpp>")
    result.append("#include <godot_cpp/variant/char_string.hpp>")
    result.append("#include <godot_cpp/variant/char_utils.hpp>")
```

### Reference Counting (`binding_generator.py:1632-1638`)

```python
if class_api["is_refcounted"]:
    # RefCounted classes need special handling
    result.append("\ttemplate <typename T>")
    result.append("\tfriend class Ref;")
    result.append("")
    result.append("\tstatic T *_gde_binding_create_callback(void *p_token, void *p_instance) {")
    result.append("\t\treturn memnew(T((GodotObject *)p_instance));")
    result.append("\t}")
```

## Build Profile System

### Profile Structure (`build_profile.py:5-64`)

Build profiles allow selective compilation of classes:

```python
def parse_build_profile(profile_filepath, api):
    """
    Profile format:
    {
        "enabled_classes": ["Node", "Node2D", "Sprite2D"],
        "disabled_classes": ["Node3D"]
    }
    """

    # Build dependency graph
    parents = {}
    children = {}
    for engine_class in api["classes"]:
        parent = engine_class.get("inherits", "")
        child = engine_class["name"]
        parents[child] = parent
        children[parent] = children.get(parent, [])
        children[parent].append(child)

    # Calculate included classes with dependencies
    return included_classes
```

### Required Dependencies (`build_profile.py:25-54`)

Certain classes are always included:

```python
# These classes are always required
REQUIRED_CLASSES = [
    "WorkerThreadPool",
    "ClassDB",
    "FileAccess",
    "Image",
    "XMLParser",
    "Semaphore"
]
```

### Method Filtering (`build_profile.py:115-139`)

Methods referencing disabled classes are excluded:

```python
def is_method_included(method, build_profile, engine_classes):
    # Check return type
    if return_type_class not in included_classes:
        return False

    # Check argument types
    for arg in method["arguments"]:
        if arg_type_class not in included_classes:
            return False

    return True
```

## Type System Mapping

### Type Conversions (`binding_generator.py:2501-2552`)

```python
def correct_typed_array(type_name):
    """Maps Godot types to C++ types"""
    if type_name.startswith("typedarray::"):
        return type_name.replace("typedarray::", "TypedArray<") + ">"
    return type_name

def correct_type(type_name, meta=None):
    """Comprehensive type correction"""
    type_conversion_map = {
        "float": "double" if precision == "double" else "float",
        "int": "int64_t",
        "bool": "bool",
        "real_t": "double" if precision == "double" else "float",
    }

    if type_name in type_conversion_map:
        return type_conversion_map[type_name]

    if is_enum(type_name):
        return get_enum_fullname(type_name)

    if is_typed_array(type_name):
        return correct_typed_array(type_name)

    return type_name
```

### Variant Type Mapping

```python
VARIANT_TYPE_MAP = {
    "Nil": "GDEXTENSION_VARIANT_TYPE_NIL",
    "bool": "GDEXTENSION_VARIANT_TYPE_BOOL",
    "int": "GDEXTENSION_VARIANT_TYPE_INT",
    "float": "GDEXTENSION_VARIANT_TYPE_FLOAT",
    "String": "GDEXTENSION_VARIANT_TYPE_STRING",
    # ... all 40+ variant types
}
```

## Performance Optimizations

### Method Binding Caching (`binding_generator.py:598-636`)

All method bindings are cached statically:

```cpp
static struct _MethodBindings {
    GDExtensionPtrConstructor constructor_0;
    GDExtensionPtrConstructor constructor_1;
    GDExtensionPtrBuiltInMethod method_length;
    GDExtensionPtrBuiltInMethod method_substr;
    // ... all methods cached
} _method_bindings;

static void init_bindings() {
    // One-time initialization
    _method_bindings.constructor_0 =
        internal::gdextension_interface_variant_get_ptr_constructor(
            GDEXTENSION_VARIANT_TYPE_STRING, 0
        );
    // ... initialize all bindings
}
```

### Inline Functions

All generated methods use forced inlining:

```cpp
_FORCE_INLINE_ int64_t length() const {
    int64_t ret;
    _method_bindings.method_length(_native_ptr(), nullptr, &ret, 0);
    return ret;
}
```

### Template Specializations (`binding_generator.py:2451-2552`)

Variadic templates for flexible argument passing:

```cpp
template <typename... Args>
Variant call(const StringName &method, Args... args) {
    std::array<Variant, sizeof...(Args)> variant_args{ Variant(args)... };
    std::array<const Variant *, sizeof...(Args)> call_args;
    for (size_t i = 0; i < sizeof...(Args); ++i) {
        call_args[i] = &variant_args[i];
    }
    // Optimized call path
}
```

## Virtual Method Generation

### GDVIRTUAL Macros (`binding_generator.py:67-200`)

Generated macros for 0-12 arguments with/without return values:

```cpp
#define GDVIRTUAL2R(m_ret, m_name, m_type1, m_type2) \
    StringName _gdvirtual_##m_name##_sn = #m_name; \
    _FORCE_INLINE_ bool _gdvirtual_##m_name##_call(m_type1 arg1, m_type2 arg2, m_ret &r_ret) const { \
        if (::godot::internal::gdextension_interface_object_has_script_method(_owner, &_gdvirtual_##m_name##_sn)) { \
            GDExtensionCallError ce; \
            Variant vargs[2] = { Variant(arg1), Variant(arg2) }; \
            const Variant *vargptrs[2] = { &vargs[0], &vargs[1] }; \
            Variant ret; \
            ::godot::internal::gdextension_interface_object_call_script_method(_owner, &_gdvirtual_##m_name##_sn, vargptrs, 2, &ret, &ce); \
            if (ce.error == GDEXTENSION_CALL_OK) { \
                r_ret = VariantCaster<m_ret>::cast(ret); \
                return true; \
            } \
        } \
        if (ClassDB::get_virtual_func(_get_class_static(), #m_name)) { \
            // Check parent virtual implementation \
        } \
        return false; \
    }
```

### Virtual Method Detection

The generator analyzes method names starting with `_` to identify virtual methods:

```python
if method["is_virtual"]:
    # Generate GDVIRTUAL macro usage
    arg_count = len(method["arguments"])
    if method.get("return_value"):
        macro_name = f"GDVIRTUAL{arg_count}R"
    else:
        macro_name = f"GDVIRTUAL{arg_count}"
```

## Documentation Generation

### Documentation Compression (`doc_source_generator.py:24-46`)

Documentation is compressed and embedded:

```python
def generate_doc_source(dst, source):
    buf = ""
    for src in source:
        with open(src, "r", encoding="utf-8") as f:
            buf += f.read()

    # Compress with best compression
    buf = zlib.compress(buf.encode("utf-8"), zlib.Z_BEST_COMPRESSION)

    # Generate C++ source with compressed data
    g.write('static const char *_doc_data_hash = "' + str(hash(buf)) + '";\n')
    g.write("static const int _doc_data_uncompressed_size = " + str(len(buf_uncompressed)) + ";\n")
    g.write("static const int _doc_data_compressed_size = " + str(len(buf)) + ";\n")
    g.write("static const unsigned char _doc_data_compressed[] = {\n")

    # Write compressed bytes
    for i, byte in enumerate(buf):
        g.write(str(byte) + ",\n" if (i % 16 == 15) else str(byte) + ", ")
```

## Error Handling

### Validation Checks

#### API Version Check (`binding_generator.py:289-295`)

```python
if api["header"]["precision"] != precision:
    print(f"Error: Precision mismatch. API uses {api['header']['precision']} "
          f"but you're targeting {precision}")
    exit(1)
```

#### Missing Method Handling

```python
if "hash" not in method:
    print(f"Warning: Method {method['name']} has no hash")
    method["hash"] = 0
```

#### Type Validation

```python
def validate_type(type_name):
    if not is_pod_type(type_name) and \
       not is_engine_class(type_name) and \
       not is_builtin_class(type_name):
        print(f"Warning: Unknown type {type_name}")
```

## Implementation Examples

### Generated Builtin Class Example

```cpp
// Generated String class
class String {
    static constexpr size_t STRING_SIZE = 8;
    uint8_t opaque[STRING_SIZE] = {};

    friend class Variant;

    static struct _MethodBindings {
        GDExtensionPtrConstructor constructor_0;
        GDExtensionPtrConstructor constructor_from_cstring;
        GDExtensionPtrBuiltInMethod method_length;
        GDExtensionPtrBuiltInMethod method_substr;
        GDExtensionPtrOperatorEvaluator operator_add_String;
    } _method_bindings;

public:
    _FORCE_INLINE_ String() {
        _method_bindings.constructor_0(_native_ptr());
    }

    _FORCE_INLINE_ String(const char *p_str) {
        _method_bindings.constructor_from_cstring(_native_ptr(), &p_str);
    }

    _FORCE_INLINE_ int64_t length() const {
        int64_t ret;
        _method_bindings.method_length(_native_ptr(), nullptr, &ret, 0);
        return ret;
    }

    _FORCE_INLINE_ String operator+(const String &p_other) const {
        String ret;
        _method_bindings.operator_add_String(
            _native_ptr(),
            p_other._native_ptr(),
            ret._native_ptr()
        );
        return ret;
    }
};
```

### Generated Engine Class Example

```cpp
// Generated Node2D class
class Node2D : public CanvasItem {
    GDCLASS(Node2D, CanvasItem)

public:
    void set_position(const Vector2 &p_position);
    Vector2 get_position() const;

    void set_rotation(double p_radians);
    double get_rotation() const;

    void set_scale(const Vector2 &p_scale);
    Vector2 get_scale() const;

    Transform2D get_transform() const;

protected:
    static void _bind_methods();

private:
    static struct _MethodBindings {
        MethodBind *method_set_position;
        MethodBind *method_get_position;
        MethodBind *method_set_rotation;
        MethodBind *method_get_rotation;
        MethodBind *method_set_scale;
        MethodBind *method_get_scale;
        MethodBind *method_get_transform;
    } _method_bindings;
};
```

### Build Profile Usage

```json
{
    "enabled_classes": [
        "Node",
        "Node2D",
        "Sprite2D",
        "AnimationPlayer"
    ],
    "disabled_classes": [
        "Node3D",
        "RenderingServer"
    ]
}
```

## Conclusion

The godot-cpp binding generator is a sophisticated code generation system that:

1. **Processes** a massive 337K+ line API definition with complete type information
2. **Generates** type-safe C++ bindings with exact memory layouts
3. **Optimizes** through static caching, forced inlining, and template specialization
4. **Handles** complex special cases including singletons, virtual methods, and type conversions
5. **Supports** build profiles for selective compilation
6. **Provides** comprehensive documentation embedding

The system demonstrates advanced code generation techniques, careful API design, and optimization strategies that enable efficient cross-language bindings while maintaining type safety and performance.
