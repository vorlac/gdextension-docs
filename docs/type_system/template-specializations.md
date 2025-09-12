# Template Specializations

## Overview

The template specialization system in godot-cpp provides compile-time type safety and efficient cross-binary marshalling through extensive use of C++ template metaprogramming. This system bridges the gap between C++'s static type system and Godot's dynamic variant-based system.

**Key Source Files:**
- `include/godot_cpp/core/type_info.hpp` - Type information templates
- `include/godot_cpp/core/binder_common.hpp` - Binding utilities and casting
- `include/godot_cpp/core/method_ptrcall.hpp` - Method pointer call marshalling
- `include/godot_cpp/variant/variant_internal.hpp` - Internal variant access
- `include/godot_cpp/classes/ref.hpp` - Reference counting templates
- `include/godot_cpp/templates/hashfuncs.hpp` - Hash function specializations

## GetTypeInfo System

### Core Template Definition

The foundation of type introspection ([type_info.hpp:103](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L103)):

```cpp
template <typename T, typename = void>
struct GetTypeInfo;
```

This primary template is never instantiated directly - all uses go through specializations.

### Type Information Structure

Each specialization provides:

```cpp
struct GetTypeInfo<SpecificType> {
    // Godot variant type this C++ type maps to
    static constexpr GDExtensionVariantType VARIANT_TYPE;

    // Additional metadata about the type
    static constexpr GDExtensionClassMethodArgumentMetadata METADATA;

    // Generate property information for the type
    static inline PropertyInfo get_class_info();
};
```

### `MAKE_TYPE_INFO` Macros

#### Basic Type Registration ([type_info.hpp:106](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L106))

```cpp
#define MAKE_TYPE_INFO(m_type, m_var_type) \
    template <> \
    struct GetTypeInfo<m_type> { \
        static constexpr GDExtensionVariantType VARIANT_TYPE = m_var_type; \
        static constexpr GDExtensionClassMethodArgumentMetadata METADATA = \
            GDEXTENSION_METHOD_ARGUMENT_METADATA_NONE; \
        static inline PropertyInfo get_class_info() { \
            return make_property_info((Variant::Type)VARIANT_TYPE, ""); \
        } \
    };
```

#### Type with Metadata ([type_info.hpp:124](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L124))

```cpp
#define MAKE_TYPE_INFO_WITH_META(m_type, m_var_type, m_metadata) \
    template <> \
    struct GetTypeInfo<m_type> { \
        static constexpr GDExtensionVariantType VARIANT_TYPE = m_var_type; \
        static constexpr GDExtensionClassMethodArgumentMetadata METADATA = m_metadata; \
        static inline PropertyInfo get_class_info() { \
            return make_property_info((Variant::Type)VARIANT_TYPE, ""); \
        } \
    };
```

### Built-in Type Specializations

#### Primitive Types ([type_info.hpp:142](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L142))

```cpp
MAKE_TYPE_INFO(bool, GDEXTENSION_VARIANT_TYPE_BOOL)
MAKE_TYPE_INFO_WITH_META(uint8_t, GDEXTENSION_VARIANT_TYPE_INT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_INT_IS_UINT8)
MAKE_TYPE_INFO_WITH_META(int8_t, GDEXTENSION_VARIANT_TYPE_INT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_INT_IS_INT8)
MAKE_TYPE_INFO_WITH_META(uint16_t, GDEXTENSION_VARIANT_TYPE_INT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_INT_IS_UINT16)
MAKE_TYPE_INFO_WITH_META(int16_t, GDEXTENSION_VARIANT_TYPE_INT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_INT_IS_INT16)
MAKE_TYPE_INFO_WITH_META(uint32_t, GDEXTENSION_VARIANT_TYPE_INT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_INT_IS_UINT32)
MAKE_TYPE_INFO_WITH_META(int32_t, GDEXTENSION_VARIANT_TYPE_INT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_INT_IS_INT32)
MAKE_TYPE_INFO_WITH_META(uint64_t, GDEXTENSION_VARIANT_TYPE_INT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_INT_IS_UINT64)
MAKE_TYPE_INFO(int64_t, GDEXTENSION_VARIANT_TYPE_INT)
```

#### Mathematical Types ([type_info.hpp:152](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L152))

```cpp
MAKE_TYPE_INFO(float, GDEXTENSION_VARIANT_TYPE_FLOAT)
MAKE_TYPE_INFO_WITH_META(double, GDEXTENSION_VARIANT_TYPE_FLOAT,
                         GDEXTENSION_METHOD_ARGUMENT_METADATA_REAL_IS_DOUBLE)
MAKE_TYPE_INFO(String, GDEXTENSION_VARIANT_TYPE_STRING)
MAKE_TYPE_INFO(Vector2, GDEXTENSION_VARIANT_TYPE_VECTOR2)
MAKE_TYPE_INFO(Vector2i, GDEXTENSION_VARIANT_TYPE_VECTOR2I)
MAKE_TYPE_INFO(Rect2, GDEXTENSION_VARIANT_TYPE_RECT2)
MAKE_TYPE_INFO(Vector3, GDEXTENSION_VARIANT_TYPE_VECTOR3)
MAKE_TYPE_INFO(Transform2D, GDEXTENSION_VARIANT_TYPE_TRANSFORM2D)
MAKE_TYPE_INFO(Vector4, GDEXTENSION_VARIANT_TYPE_VECTOR4)
MAKE_TYPE_INFO(Plane, GDEXTENSION_VARIANT_TYPE_PLANE)
MAKE_TYPE_INFO(Quaternion, GDEXTENSION_VARIANT_TYPE_QUATERNION)
MAKE_TYPE_INFO(AABB, GDEXTENSION_VARIANT_TYPE_AABB)
MAKE_TYPE_INFO(Basis, GDEXTENSION_VARIANT_TYPE_BASIS)
MAKE_TYPE_INFO(Transform3D, GDEXTENSION_VARIANT_TYPE_TRANSFORM3D)
MAKE_TYPE_INFO(Projection, GDEXTENSION_VARIANT_TYPE_PROJECTION)
```

### Object Pointer Specialization

SFINAE-based specialization for Object-derived pointers ([type_info.hpp:210](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L210)):

```cpp
template <typename T>
struct GetTypeInfo<T *, typename EnableIf<TypeInherits<Object, T>::value>::type> {
    static constexpr GDExtensionVariantType VARIANT_TYPE =
        GDEXTENSION_VARIANT_TYPE_OBJECT;
    static constexpr GDExtensionClassMethodArgumentMetadata METADATA =
        GDEXTENSION_METHOD_ARGUMENT_METADATA_NONE;

    static inline PropertyInfo get_class_info() {
        return make_property_info(
            Variant::Type::OBJECT,
            "",
            PROPERTY_HINT_RESOURCE_TYPE,
            T::get_class_static()
        );
    }
};

// Const pointer specialization
template <typename T>
struct GetTypeInfo<const T *,
    typename EnableIf<TypeInherits<Object, T>::value>::type> {
    // Same implementation for const pointers
};
```

### Enum Support System

#### Enum Type Registration ([type_info.hpp:237](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L237))

```cpp
#define MAKE_ENUM_TYPE_INFO(m_enum) \
    TEMPL_MAKE_ENUM_TYPE_INFO(m_enum, m_enum) \
    TEMPL_MAKE_ENUM_TYPE_INFO(m_enum, m_enum const) \
    TEMPL_MAKE_ENUM_TYPE_INFO(m_enum, m_enum &) \
    TEMPL_MAKE_ENUM_TYPE_INFO(m_enum, const m_enum &)

#define TEMPL_MAKE_ENUM_TYPE_INFO(m_enum, m_qualified) \
    template <> \
    struct GetTypeInfo<m_qualified> { \
        static constexpr Variant::Type VARIANT_TYPE = Variant::INT; \
        static constexpr GDExtensionClassMethodArgumentMetadata METADATA = \
            GDEXTENSION_METHOD_ARGUMENT_METADATA_NONE; \
        static inline PropertyInfo get_class_info() { \
            return make_property_info(Variant::Type::INT, "", \
                PROPERTY_HINT_NONE, "", PROPERTY_USAGE_DEFAULT, \
                get_enum_class_info(#m_enum)); \
        } \
    };
```

### BitField Template ([type_info.hpp:275](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L275))

```cpp
template <typename T>
class BitField {
    int64_t value = 0;

public:
    _FORCE_INLINE_ void set_flag(T p_flag) { value |= p_flag; }
    _FORCE_INLINE_ bool has_flag(T p_flag) const { return value & p_flag; }
    _FORCE_INLINE_ void clear_flag(T p_flag) { value &= ~p_flag; }
    _FORCE_INLINE_ BitField(int64_t p_value) { value = p_value; }
    _FORCE_INLINE_ operator int64_t() const { return value; }
    _FORCE_INLINE_ operator Variant() const { return value; }
};

// BitField specialization
#define MAKE_BITFIELD_TYPE_INFO(m_enum) \
    template <> \
    struct GetTypeInfo<BitField<m_enum>> { \
        static constexpr Variant::Type VARIANT_TYPE = Variant::INT; \
        static constexpr GDExtensionClassMethodArgumentMetadata METADATA = \
            GDEXTENSION_METHOD_ARGUMENT_METADATA_NONE; \
        static inline PropertyInfo get_class_info() { \
            return make_property_info(Variant::Type::INT, "", \
                PROPERTY_HINT_FLAGS, "", PROPERTY_USAGE_DEFAULT, \
                get_enum_class_info(#m_enum)); \
        } \
    };
```

### TypedArray Specializations

#### Generic TypedArray ([type_info.hpp:330](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L330))

```cpp
template <typename T>
struct GetTypeInfo<TypedArray<T>> {
    static constexpr GDExtensionVariantType VARIANT_TYPE =
        GDEXTENSION_VARIANT_TYPE_ARRAY;
    static constexpr GDExtensionClassMethodArgumentMetadata METADATA =
        GDEXTENSION_METHOD_ARGUMENT_METADATA_NONE;

    static inline PropertyInfo get_class_info() {
        return make_property_info(
            Variant::Type::ARRAY,
            "",
            PROPERTY_HINT_ARRAY_TYPE,
            T::get_class_static()
        );
    }
};
```

#### Built-in TypedArray Specializations ([type_info.hpp:359](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L359))

```cpp
#define MAKE_TYPED_ARRAY_INFO(m_type, m_variant_type) \
    template <> \
    struct GetTypeInfo<TypedArray<m_type>> { \
        static constexpr GDExtensionVariantType VARIANT_TYPE = \
            GDEXTENSION_VARIANT_TYPE_ARRAY; \
        static constexpr GDExtensionClassMethodArgumentMetadata METADATA = \
            GDEXTENSION_METHOD_ARGUMENT_METADATA_NONE; \
        static inline PropertyInfo get_class_info() { \
            return make_property_info(Variant::Type::ARRAY, "", \
                PROPERTY_HINT_ARRAY_TYPE, \
                Variant::get_type_name(m_variant_type).utf8().get_data()); \
        } \
    };

MAKE_TYPED_ARRAY_INFO(bool, Variant::BOOL)
MAKE_TYPED_ARRAY_INFO(uint8_t, Variant::INT)
MAKE_TYPED_ARRAY_INFO(int32_t, Variant::INT)
MAKE_TYPED_ARRAY_INFO(int64_t, Variant::INT)
MAKE_TYPED_ARRAY_INFO(float, Variant::FLOAT)
MAKE_TYPED_ARRAY_INFO(double, Variant::FLOAT)
MAKE_TYPED_ARRAY_INFO(String, Variant::STRING)
// ... more types
```

## Type Traits and SFINAE

### Type Inheritance Detection ([type_info.hpp:71](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L71))

```cpp
template <typename B, typename D>
struct TypeInherits {
    static D *get_d();

    // SFINAE: test(B*) is selected if D* converts to B*
    static char (&test(B *))[1];
    static char (&test(...))[2];

    static bool const value = sizeof(test(get_d())) == sizeof(char) &&
        !TypesAreSame<B volatile const, void volatile const>::value;
};
```

### EnableIf Implementation ([type_info.hpp:42](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L42))

```cpp
template <bool C, typename T = void>
struct EnableIf {
    typedef T type;
};

template <typename T>
struct EnableIf<false, T> {
    // No 'type' member when condition is false
};
```

### Type Comparison ([type_info.hpp:51](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L51))

```cpp
template <typename, typename>
struct TypesAreSame {
    static bool const value = false;
};

template <typename A>
struct TypesAreSame<A, A> {
    static bool const value = true;
};
```

### Function Comparison ([type_info.hpp:61](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/type_info.hpp#L61))

```cpp
template <auto A, auto B>
struct FunctionsAreSame {
    static bool const value = false;
};

template <auto A>
struct FunctionsAreSame<A, A> {
    static bool const value = true;
};
```

## VariantCaster Templates

### Base Template ([binder_common.hpp:86](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L86))

```cpp
template <typename T>
struct VariantCaster {
    static _FORCE_INLINE_ T cast(const Variant &p_variant) {
        using TStripped = std::remove_pointer_t<T>;

        // Compile-time branch using if constexpr (C++17)
        if constexpr (std::is_base_of<Object, TStripped>::value) {
            return Object::cast_to<TStripped>(p_variant);
        } else {
            return p_variant;  // Implicit conversion operator
        }
    }
};
```

### Reference Specializations ([binder_common.hpp:98](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L98))

```cpp
template <typename T>
struct VariantCaster<T &> {
    static _FORCE_INLINE_ T cast(const Variant &p_variant) {
        using TStripped = std::remove_pointer_t<T>;
        if constexpr (std::is_base_of<Object, TStripped>::value) {
            return Object::cast_to<TStripped>(p_variant);
        } else {
            return p_variant;
        }
    }
};

template <typename T>
struct VariantCaster<const T &> {
    static _FORCE_INLINE_ T cast(const Variant &p_variant) {
        using TStripped = std::remove_pointer_t<T>;
        if constexpr (std::is_base_of<Object, TStripped>::value) {
            return Object::cast_to<TStripped>(p_variant);
        } else {
            return p_variant;
        }
    }
};
```

### Object Class Checker ([binder_common.hpp:122](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L122))

```cpp
template <typename T>
struct VariantObjectClassChecker {
    static _FORCE_INLINE_ bool check(const Variant &p_variant) {
        using TStripped = std::remove_pointer_t<T>;
        if constexpr (std::is_base_of<Object, TStripped>::value) {
            Object *obj = p_variant;
            return Object::cast_to<TStripped>(p_variant) || !obj;
        } else {
            return true;  // Non-objects always pass
        }
    }
};

// Ref<T> specialization
template <typename T>
struct VariantObjectClassChecker<const Ref<T> &> {
    static _FORCE_INLINE_ bool check(const Variant &p_variant) {
        Object *obj = p_variant;
        const Ref<T> node = p_variant;
        return node.ptr() || !obj;
    }
};
```

### Validation and Casting ([binder_common.hpp:147](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L147))

```cpp
template <typename T>
struct VariantCasterAndValidate {
    static _FORCE_INLINE_ T cast(const Variant **p_args,
                                 uint32_t p_arg_idx,
                                 GDExtensionCallError &r_error) {
        GDExtensionVariantType argtype =
            GDExtensionVariantType(GetTypeInfo<T>::VARIANT_TYPE);

        if (!internal::gdextension_interface_variant_can_convert_strict(
                static_cast<GDExtensionVariantType>(p_args[p_arg_idx]->get_type()),
                argtype) ||
            !VariantObjectClassChecker<T>::check(p_args[p_arg_idx])) {

            r_error.error = GDEXTENSION_CALL_ERROR_INVALID_ARGUMENT;
            r_error.argument = p_arg_idx;
            r_error.expected = argtype;
        }

        return VariantCaster<T>::cast(*p_args[p_arg_idx]);
    }
};
```

### Enum Casting Macro ([binder_common.hpp:44](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L44))

```cpp
#define VARIANT_ENUM_CAST(m_enum) \
namespace godot { \
    MAKE_ENUM_TYPE_INFO(m_enum) \
    template <> \
    struct VariantCaster<m_enum> { \
        static _FORCE_INLINE_ m_enum cast(const Variant &p_variant) { \
            return (m_enum)p_variant.operator int64_t(); \
        } \
    }; \
    template <> \
    struct PtrToArg<m_enum> { \
        _FORCE_INLINE_ static m_enum convert(const void *p_ptr) { \
            return m_enum(*reinterpret_cast<const int64_t *>(p_ptr)); \
        } \
        typedef int64_t EncodeT; \
        _FORCE_INLINE_ static void encode(m_enum p_val, void *p_ptr) { \
            *reinterpret_cast<int64_t *>(p_ptr) = p_val; \
        } \
    }; \
}
```

## PtrToArg Marshalling System

### Base Template ([method_ptrcall.hpp:41](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/method_ptrcall.hpp#L41))

```cpp
template <typename T>
struct PtrToArg {};  // Primary template is empty
```

### Basic Type Marshalling ([method_ptrcall.hpp:44](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/method_ptrcall.hpp#L44))

```cpp
#define MAKE_PTRARG(m_type) \
    template <> \
    struct PtrToArg<m_type> { \
        _FORCE_INLINE_ static m_type convert(const void *p_ptr) { \
            return *reinterpret_cast<const m_type *>(p_ptr); \
        } \
        typedef m_type EncodeT; \
        _FORCE_INLINE_ static void encode(m_type p_val, void *p_ptr) { \
            *reinterpret_cast<m_type *>(p_ptr) = p_val; \
        } \
    };

MAKE_PTRARG(bool)
MAKE_PTRARG(uint8_t)
MAKE_PTRARG(int8_t)
MAKE_PTRARG(uint16_t)
MAKE_PTRARG(int16_t)
MAKE_PTRARG(uint32_t)
MAKE_PTRARG(int64_t)
MAKE_PTRARG(float)
MAKE_PTRARG(double)
```

### Type Conversion Marshalling ([method_ptrcall.hpp:64](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/method_ptrcall.hpp#L64))

```cpp
#define MAKE_PTRARGCONV(m_type, m_conv) \
    template <> \
    struct PtrToArg<m_type> { \
        _FORCE_INLINE_ static m_type convert(const void *p_ptr) { \
            return static_cast<m_type>(*reinterpret_cast<const m_conv *>(p_ptr)); \
        } \
        typedef m_conv EncodeT; \
        _FORCE_INLINE_ static void encode(m_type p_val, void *p_ptr) { \
            *reinterpret_cast<m_conv *>(p_ptr) = static_cast<m_conv>(p_val); \
        } \
        _FORCE_INLINE_ static m_conv encode_arg(m_type p_val) { \
            return static_cast<m_conv>(p_val); \
        } \
    };

// Integer conversions
MAKE_PTRARGCONV(uint32_t, int64_t)
MAKE_PTRARGCONV(int32_t, int64_t)
```

### Object Pointer Marshalling ([method_ptrcall.hpp:172](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/method_ptrcall.hpp#L172))

```cpp
template <typename T>
struct PtrToArg<T *> {
    static_assert(std::is_base_of<Object, T>::value,
                  "Cannot encode non-Object value as an Object");

    _FORCE_INLINE_ static T *convert(const void *p_ptr) {
        return likely(p_ptr) ?
            reinterpret_cast<T *>(
                godot::internal::get_object_instance_binding(
                    *reinterpret_cast<GDExtensionObjectPtr *>(
                        const_cast<void *>(p_ptr)
                    )
                )
            ) : nullptr;
    }

    typedef Object *EncodeT;

    _FORCE_INLINE_ static void encode(T *p_var, void *p_ptr) {
        *reinterpret_cast<const void **>(p_ptr) =
            likely(p_var) ? p_var->_owner : nullptr;
    }
};
```

### Native Pointer Support ([method_ptrcall.hpp:197](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/method_ptrcall.hpp#L197))

```cpp
#define GDVIRTUAL_NATIVE_PTR(m_type) \
    template <> \
    struct PtrToArg<m_type *> { \
        _FORCE_INLINE_ static m_type *convert(const void *p_ptr) { \
            return (m_type *)(*(void **)p_ptr); \
        } \
        typedef m_type *EncodeT; \
        _FORCE_INLINE_ static void encode(m_type *p_var, void *p_ptr) { \
            *reinterpret_cast<m_type **>(p_ptr) = p_var; \
        } \
    };

// Applied to native types
GDVIRTUAL_NATIVE_PTR(void)
GDVIRTUAL_NATIVE_PTR(AudioFrame)
GDVIRTUAL_NATIVE_PTR(bool)
GDVIRTUAL_NATIVE_PTR(char)
// ... more native types
```

## Method Call Dispatch

### Index Sequence Generation ([defs.hpp:300](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/defs.hpp#L300))

```cpp
template <size_t... Is>
struct IndexSequence {};

template <size_t N, size_t... Is>
struct BuildIndexSequence : BuildIndexSequence<N - 1, N - 1, Is...> {};

template <size_t... Is>
struct BuildIndexSequence<0, Is...> : IndexSequence<Is...> {};
```

### Variadic Method Call ([binder_common.hpp:192](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L192))

```cpp
// Non-const method, no return
template <typename T, typename... P, size_t... Is>
void call_with_ptr_args_helper(T *p_instance,
                              void (T::*p_method)(P...),
                              const GDExtensionConstTypePtr *p_args,
                              IndexSequence<Is...>) {
    (p_instance->*p_method)(PtrToArg<P>::convert(p_args[Is])...);
}

template <typename T, typename... P>
void call_with_ptr_args(T *p_instance,
                        void (T::*p_method)(P...),
                        const GDExtensionConstTypePtr *p_args,
                        void *) {
    call_with_ptr_args_helper(p_instance, p_method, p_args,
                             BuildIndexSequence<sizeof...(P)>{});
}

// With return value
template <typename T, typename R, typename... P, size_t... Is>
void call_with_ptr_args_ret_helper(T *p_instance,
                                   R (T::*p_method)(P...),
                                   const GDExtensionConstTypePtr *p_args,
                                   void *r_ret,
                                   IndexSequence<Is...>) {
    PtrToArg<R>::encode((p_instance->*p_method)(PtrToArg<P>::convert(p_args[Is])...),
                        r_ret);
}
```

### Static Method Dispatch ([binder_common.hpp:268](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L268))

```cpp
template <typename... P, size_t... Is>
void call_with_ptr_args_static_helper(void (*p_method)(P...),
                                      const GDExtensionConstTypePtr *p_args,
                                      IndexSequence<Is...>) {
    p_method(PtrToArg<P>::convert(p_args[Is])...);
}

template <typename R, typename... P, size_t... Is>
void call_with_ptr_args_static_ret_helper(R (*p_method)(P...),
                                          const GDExtensionConstTypePtr *p_args,
                                          void *r_ret,
                                          IndexSequence<Is...>) {
    PtrToArg<R>::encode(p_method(PtrToArg<P>::convert(p_args[Is])...), r_ret);
}
```

### Argument Type Extraction ([binder_common.hpp:480](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/core/binder_common.hpp#L480))

```cpp
template <typename Q>
void call_get_argument_type_helper(int p_arg, int &index,
                                   GDExtensionVariantType &type) {
    if (p_arg == index) {
        type = GDExtensionVariantType(GetTypeInfo<Q>::VARIANT_TYPE);
    }
    index++;
}

template <typename... P>
GDExtensionVariantType call_get_argument_type(int p_arg) {
    GDExtensionVariantType type = GDEXTENSION_VARIANT_TYPE_NIL;
    int index = 0;

    // Parameter pack expansion trick
    using expand_type = int[];
    expand_type a{ 0, (call_get_argument_type_helper<P>(p_arg, index, type), 0)... };
    (void)a;  // Suppress unused variable warning

    return type;
}
```

## Reference Type Handling

### Ref<T> Template Core ([ref.hpp:47](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/templates/ref.hpp#L47))

```cpp
template <typename T>
class Ref {
    T *reference = nullptr;

    void ref(const Ref &p_from) {
        if (p_from.reference == reference) {
            return;
        }

        unref();
        reference = p_from.reference;
        if (reference) {
            reference->reference();  // Increment ref count
        }
    }

    void ref_pointer(T *p_ref) {
        ERR_FAIL_NULL(p_ref);

        if (p_ref->init_ref()) {
            reference = p_ref;
        }
    }

    void unref() {
        if (reference && reference->unreference()) {
            memdelete(reference);
        }
        reference = nullptr;
    }

public:
    // Smart pointer operations
    inline T *operator->() { return reference; }
    inline const T *operator->() const { return reference; }
    inline T *ptr() { return reference; }
    inline const T *ptr() const { return reference; }
};
```

### Ref<T> PtrToArg Specialization ([ref.hpp:232](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/templates/ref.hpp#L232))

```cpp
template <typename T>
struct PtrToArg<Ref<T>> {
    _FORCE_INLINE_ static Ref<T> convert(const void *p_ptr) {
        GDExtensionRefPtr ref = (GDExtensionRefPtr)p_ptr;
        if (unlikely(!p_ptr)) {
            return Ref<T>();
        }

        // Extract object from GDExtension ref
        return Ref<T>(reinterpret_cast<T *>(
            godot::internal::get_object_instance_binding(
                godot::internal::gdextension_interface_ref_get_object(ref)
            )
        ));
    }

    typedef Ref<T> EncodeT;

    _FORCE_INLINE_ static void encode(Ref<T> p_val, void *p_ptr) {
        GDExtensionRefPtr ref = (GDExtensionRefPtr)p_ptr;
        ERR_FAIL_NULL(ref);

        // Set object in GDExtension ref
        if (p_val.is_valid()) {
            godot::internal::gdextension_interface_ref_set_object(
                ref,
                p_val->_owner
            );
        }
    }
};
```

### Ref<T> GetTypeInfo Specialization ([ref.hpp:269](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/templates/ref.hpp#L269))

```cpp
template <typename T>
struct GetTypeInfo<Ref<T>,
    typename EnableIf<TypeInherits<RefCounted, T>::value>::type> {

    static constexpr GDExtensionVariantType VARIANT_TYPE =
        GDEXTENSION_VARIANT_TYPE_OBJECT;
    static constexpr GDExtensionClassMethodArgumentMetadata METADATA =
        GDEXTENSION_METHOD_ARGUMENT_METADATA_NONE;

    static inline PropertyInfo get_class_info() {
        return make_property_info(
            Variant::Type::OBJECT,
            "",
            PROPERTY_HINT_RESOURCE_TYPE,
            T::get_class_static()
        );
    }
};
```

## Container Specializations

### TypedArray Implementation ([typed_array.hpp:42](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/typed_array.hpp#L42))

```cpp
template <typename T>
class TypedArray : public Array {
public:
    _FORCE_INLINE_ void set_typed(uint32_t p_type,
                                  const StringName &p_class_name,
                                  const Variant &p_script) {
        // Set array type constraints
        static_cast<Array *>(this)->set_typed(p_type, p_class_name, p_script);
    }

    _FORCE_INLINE_ void set_typed(const Variant::Type p_variant_type) {
        // Convenience method for built-in types
        set_typed(p_variant_type, StringName(), Variant());
    }

    _FORCE_INLINE_ TypedArray(const Variant &p_variant) :
        Array(Array(p_variant),
              GetTypeInfo<T>::VARIANT_TYPE,
              get_class_name(),
              Variant()) {
    }

    _FORCE_INLINE_ TypedArray(const Array &p_array) :
        Array(p_array,
              GetTypeInfo<T>::VARIANT_TYPE,
              get_class_name(),
              Variant()) {
    }

    _FORCE_INLINE_ TypedArray() {
        set_typed(GetTypeInfo<T>::VARIANT_TYPE, get_class_name(), Variant());
    }
};
```

### TypedDictionary System ([typed_dictionary.hpp:40](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/typed_dictionary.hpp#L40))

Extensive macro-based specialization for all type combinations:

```cpp
#define MAKE_TYPED_DICTIONARY_EXPANDED(m_type_key, m_variant_type_key, \
                                       m_type_value, m_variant_type_value) \
template <> \
class TypedDictionary<m_type_key, m_type_value> : public Dictionary { \
public: \
    _FORCE_INLINE_ TypedDictionary() : \
        Dictionary(Dictionary::with_typed_keys_values( \
            m_variant_type_key, m_variant_type_value)) {} \
    \
    _FORCE_INLINE_ TypedDictionary(const Variant &p_variant) : \
        Dictionary(Dictionary(p_variant).with_typed_keys_values( \
            m_variant_type_key, m_variant_type_value)) {} \
    \
    _FORCE_INLINE_ TypedDictionary(const Dictionary &p_dictionary) : \
        Dictionary(p_dictionary.with_typed_keys_values( \
            m_variant_type_key, m_variant_type_value)) {} \
};
```

## Hash Function Templates

### HashMapHasherDefault ([hashfuncs.hpp:314](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/hashfuncs.hpp#L314))

```cpp
struct HashMapHasherDefault {
    // Pointer hashing
    template <typename T>
    static _FORCE_INLINE_ uint32_t hash(const T *p_pointer) {
        return hash_one_uint64((uint64_t)p_pointer);
    }

    // Ref<T> hashing
    template <typename T>
    static _FORCE_INLINE_ uint32_t hash(const Ref<T> &p_ref) {
        return hash_one_uint64((uint64_t)p_ref.operator->());
    }

    // String hashing
    static _FORCE_INLINE_ uint32_t hash(const String &p_string) {
        return p_string.hash();
    }

    // StringName hashing
    static _FORCE_INLINE_ uint32_t hash(const StringName &p_string_name) {
        return p_string_name.hash();
    }

    // Integer hashing
    static _FORCE_INLINE_ uint32_t hash(const uint64_t p_int) {
        return hash_one_uint64(p_int);
    }

    static _FORCE_INLINE_ uint32_t hash(const int64_t p_int) {
        return hash_one_uint64((uint64_t)p_int);
    }

    // Float hashing (handles NaN)
    static _FORCE_INLINE_ uint32_t hash(const float p_float) {
        return hash_murmur3_one_float(p_float);
    }

    static _FORCE_INLINE_ uint32_t hash(const double p_double) {
        return hash_murmur3_one_double(p_double);
    }

    // Object hashing
    static _FORCE_INLINE_ uint32_t hash(const Object *p_object) {
        return hash_one_uint64((uint64_t)p_object);
    }
};
```

### Specialized Comparators ([hashfuncs.hpp:418](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/hashfuncs.hpp#L418))

```cpp
// Float comparison with NaN handling
template <>
struct HashMapComparatorDefault<float> {
    static bool compare(const float &p_lhs, const float &p_rhs) {
        return (p_lhs == p_rhs) || (Math::is_nan(p_lhs) && Math::is_nan(p_rhs));
    }
};

template <>
struct HashMapComparatorDefault<double> {
    static bool compare(const double &p_lhs, const double &p_rhs) {
        return (p_lhs == p_rhs) || (Math::is_nan(p_lhs) && Math::is_nan(p_rhs));
    }
};

// Vector comparison
template <>
struct HashMapComparatorDefault<Vector2> {
    static bool compare(const Vector2 &p_lhs, const Vector2 &p_rhs) {
        return ((p_lhs.x == p_rhs.x) ||
                (Math::is_nan(p_lhs.x) && Math::is_nan(p_rhs.x))) &&
               ((p_lhs.y == p_rhs.y) ||
                (Math::is_nan(p_lhs.y) && Math::is_nan(p_rhs.y)));
    }
};
```

## Cross-Binary Type Safety

### Variant Internal Type Mapping ([variant_internal.hpp:40](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/variant/variant_internal.hpp#L40))

```cpp
template <typename T>
struct VariantInternalType {};

// Specializations for all types
template <>
struct VariantInternalType<bool> {
    static constexpr Variant::Type type = Variant::BOOL;
};

template <>
struct VariantInternalType<int64_t> {
    static constexpr Variant::Type type = Variant::INT;
};

template <>
struct VariantInternalType<double> {
    static constexpr Variant::Type type = Variant::FLOAT;
};

// ... specializations for all 40 variant types
```

### Internal Value Access ([variant_internal.hpp:474](https://github.com/godotengine/godot-cpp/blob/master/include/godot_cpp/variant/variant_internal.hpp#L474))

```cpp
template <typename T>
struct VariantInternalAccessor {
    static _FORCE_INLINE_ const T &get(const Variant *v) {
        return *VariantInternal::get_internal_value<T>(v);
    }

    template <typename U = T,
              typename = std::enable_if_t<can_set_variant_internal_value<U>::value>>
    static _FORCE_INLINE_ void set(Variant *v,
                                   const internal::VariantInternalType<U> &p_value) {
        *VariantInternal::get_internal_value<U>(v) = p_value;
    }
};

// Specialization for Object
template <>
struct VariantInternalAccessor<Object *> {
    static _FORCE_INLINE_ Object *get(const Variant *v) {
        return *VariantInternal::get_object(v);
    }

    static _FORCE_INLINE_ void set(Variant *v, const Object *p_value) {
        *VariantInternal::get_object(v) = const_cast<Object *>(p_value);
    }
};
```

## Compile-Time Validation

### Static Assertions in Templates

```cpp
// Object pointer validation
template <typename T>
struct PtrToArg<T *> {
    static_assert(std::is_base_of<Object, T>::value,
                  "Cannot encode non-Object value as an Object");
    // ...
};

// Class declaration validation (wrapped.hpp)
static_assert(TypesAreSame<typename T::self_type, T>::value,
              "Class not declared properly, please use GDCLASS.");

// Method binding validation
static_assert(!FunctionsAreSame<T::self_type::_bind_methods,
                                T::parent_type::_bind_methods>::value,
              "Class must declare 'static void _bind_methods'.");
```

### SFINAE-based Validation

```cpp
// Only enable for Object-derived types
template <typename T>
struct GetTypeInfo<T *,
    typename EnableIf<TypeInherits<Object, T>::value>::type> {
    // ...
};

// Only enable for RefCounted-derived types
template <typename T>
struct GetTypeInfo<Ref<T>,
    typename EnableIf<TypeInherits<RefCounted, T>::value>::type> {
    // ...
};
```

## Performance Optimizations

### Compile-Time Optimizations

1. **Constant Folding**: All VARIANT_TYPE and METADATA values are constexpr
2. **Template Instantiation**: Types resolved at compile time
3. **Inline Expansion**: _FORCE_INLINE_ used throughout for zero-cost abstractions
4. **Dead Code Elimination**: if constexpr enables compile-time branching

### Runtime Optimizations

1. **Direct Memory Access**: PtrToArg enables direct pointer manipulation
2. **Type Caching**: GetTypeInfo results are compile-time constants
3. **Move Semantics**: Ref<T> supports efficient move operations
4. **Parameter Pack Expansion**: Zero-overhead variadic forwarding

### Memory Layout Optimizations

```cpp
// Efficient parameter pack expansion
template <typename... P>
void method(P... args) {
    // Single allocation for all arguments
    using expand_type = int[];
    expand_type a{ 0, (process(args), 0)... };
}

// Direct pointer casting for POD types
template <>
struct PtrToArg<int64_t> {
    _FORCE_INLINE_ static int64_t convert(const void *p_ptr) {
        return *reinterpret_cast<const int64_t *>(p_ptr);  // No allocation
    }
};
```

## Implementation Examples

### Custom Type Registration

```cpp
// Register custom enum
enum class MyEnum {
    OPTION_A,
    OPTION_B,
    OPTION_C
};

VARIANT_ENUM_CAST(MyEnum);  // Generates all necessary specializations

// Register custom class
class MyClass : public Resource {
    GDCLASS(MyClass, Resource)

public:
    void my_method(MyEnum option, const TypedArray<Node> &nodes) {
        // Type-safe parameters
    }

    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("my_method", "option", "nodes"),
                            &MyClass::my_method);
    }
};
```

### Type-Safe Container Usage

```cpp
// Type-safe array
TypedArray<Node2D> nodes;
nodes.push_back(memnew(Node2D));  // Compile-time type checking

// Type-safe dictionary
TypedDictionary<String, Ref<Texture2D>> textures;
textures[String("player")] = load_texture("player.png");

// Automatic type conversion in methods
void process_nodes(const TypedArray<Node> &nodes) {
    for (int i = 0; i < nodes.size(); i++) {
        Node *node = Object::cast_to<Node>(nodes[i]);  // Safe cast
        if (node) {
            node->queue_free();
        }
    }
}
```

### Performance-Critical Code

```cpp
// Fast path for known types
template <typename T>
void process_variant_fast(const Variant &v) {
    if (v.get_type() == GetTypeInfo<T>::VARIANT_TYPE) {
        // Direct access without conversion
        const T *value = VariantInternal::get_internal_value<T>(&v);
        process_direct(*value);
    } else {
        // Fallback to conversion
        process_converted(v.operator T());
    }
}

// Zero-overhead method binding
template <typename R, typename... Args>
R call_method_direct(Object *obj, const StringName &method, Args... args) {
    // Direct pointer call without variant conversion
    typedef R (Object::*MethodPtr)(Args...);
    MethodPtr ptr = /* get method pointer */;
    return (obj->*ptr)(args...);
}
```

## Conclusion

The template specialization system in godot-cpp represents a sophisticated use of C++ template metaprogramming to achieve:

1. **Complete Type Safety**: Every type conversion is validated at compile time
2. **Zero-Cost Abstractions**: Template specializations compile to optimal code
3. **Seamless Integration**: C++ types map transparently to Godot's type system
4. **Extensibility**: New types can be integrated through simple macro invocations
5. **Performance**: Direct memory manipulation where possible, conversions only when necessary

This system enables godot-cpp to provide a native C++ experience while maintaining full compatibility with Godot's dynamic type system and ensuring optimal performance across the binary boundary.
