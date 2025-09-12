# Utility Functions Reference

## Introduction

**Essential helper functions for GDExtension development:** This reference covers utility functions, helper macros, and convenience classes that make GDExtension development more productive. These utilities handle common tasks like string manipulation, math operations, type conversion, and debugging.

## UtilityFunctions Class

**Core utility functions for common operations:** The UtilityFunctions class provides static methods that correspond to GDScript's built-in functions, making them available in C++.

### Debug and Logging Functions

```cpp
#include <godot_cpp/variant/utility_functions.hpp>

class UtilityExample : public RefCounted {
    GDCLASS(UtilityExample, RefCounted)

public:
    void demonstrate_logging() {
        // Basic printing
        UtilityFunctions::print("Hello World");
        UtilityFunctions::print("Value:", 42, "Text:", "example");

        // Formatted printing
        String formatted = vformat("Player %s has %d health", "Alice", 100);
        UtilityFunctions::print(formatted);

        // Print with rich text (if supported)
        UtilityFunctions::print_rich("[color=red]Error:[/color] Something went wrong");

        // Debug information
        UtilityFunctions::print("Current time:", OS::get_singleton()->get_unix_time());

        // Conditional printing
        bool debug_mode = true;
        if (debug_mode) {
            UtilityFunctions::print("Debug: Function called");
        }
    }

    void demonstrate_assertions() {
        // Assertions for debugging
        int value = 42;
        assert(value > 0);  // Standard C++ assert

        // Godot's assertion equivalent
        CRASH_COND(value < 0);  // Crashes if condition is true

        // Non-fatal error checking
        ERR_FAIL_COND(value < 0);  // Returns from function if condition is true
        ERR_FAIL_COND_V(value < 0, -1);  // Returns value if condition is true
    }
};
```

### String Utilities

```cpp
class StringUtilities : public RefCounted {
    GDCLASS(StringUtilities, RefCounted)

public:
    void demonstrate_string_functions() {
        String text = "Hello World";

        // String manipulation via UtilityFunctions
        String upper = text.to_upper();
        String lower = text.to_lower();

        // String formatting
        String formatted = vformat("Value: %d, Float: %.2f", 42, 3.14159);
        UtilityFunctions::print(formatted);

        // String conversion utilities
        String number_str = "12345";
        int number = number_str.to_int();
        float decimal = String("3.14").to_float();

        // String validation
        String empty_check = "";
        if (empty_check.is_empty()) {
            UtilityFunctions::print("String is empty");
        }

        // Path utilities
        String path = "res://assets/textures/player.png";
        String directory = path.get_base_dir();  // "res://assets/textures"
        String filename = path.get_file();       // "player.png"
        String extension = path.get_extension();  // "png"

        UtilityFunctions::print("Directory:", directory);
        UtilityFunctions::print("Filename:", filename);
        UtilityFunctions::print("Extension:", extension);
    }

    void demonstrate_advanced_strings() {
        // Regular expressions (via RegEx class)
        Ref<RegEx> regex;
        regex.instantiate();
        regex->compile(R"(\b\w+@\w+\.\w+\b)");  // Email pattern

        String text = "Contact us at support@example.com or help@test.org";
        Array matches = regex->search_all(text);

        for (int i = 0; i < matches.size(); i++) {
            Ref<RegExMatch> match = matches[i];
            UtilityFunctions::print("Found email:", match->get_string());
        }

        // String splitting and joining
        String csv = "apple,banana,cherry";
        PackedStringArray fruits = csv.split(",");

        for (int i = 0; i < fruits.size(); i++) {
            UtilityFunctions::print("Fruit:", fruits[i]);
        }

        String rejoined = String(",").join(fruits);
        UtilityFunctions::print("Rejoined:", rejoined);
    }
};
```

### Math Utilities

```cpp
class MathUtilities : public RefCounted {
    GDCLASS(MathUtilities, RefCounted)

public:
    void demonstrate_math_functions() {
        // Basic math functions
        float value = -3.7f;

        UtilityFunctions::print("abs(-3.7):", UtilityFunctions::abs(value));
        UtilityFunctions::print("sign(-3.7):", UtilityFunctions::sign(value));
        UtilityFunctions::print("floor(-3.7):", UtilityFunctions::floor(value));
        UtilityFunctions::print("ceil(-3.7):", UtilityFunctions::ceil(value));
        UtilityFunctions::print("round(-3.7):", UtilityFunctions::round(value));

        // Trigonometric functions
        float angle = Math_PI / 4.0f;  // 45 degrees
        UtilityFunctions::print("sin(45°):", UtilityFunctions::sin(angle));
        UtilityFunctions::print("cos(45°):", UtilityFunctions::cos(angle));
        UtilityFunctions::print("tan(45°):", UtilityFunctions::tan(angle));

        // Logarithmic functions
        UtilityFunctions::print("log(10):", UtilityFunctions::log(10.0));
        UtilityFunctions::print("exp(1):", UtilityFunctions::exp(1.0));
        UtilityFunctions::print("pow(2, 8):", UtilityFunctions::pow(2.0, 8.0));
        UtilityFunctions::print("sqrt(16):", UtilityFunctions::sqrt(16.0));

        // Range functions
        float clamped = UtilityFunctions::clamp(15.0f, 0.0f, 10.0f);  // 10.0
        float lerped = UtilityFunctions::lerp(0.0f, 100.0f, 0.75f);   // 75.0
        float mapped = UtilityFunctions::remap(5.0f, 0.0f, 10.0f, 100.0f, 200.0f);  // 150.0

        UtilityFunctions::print("clamp(15, 0, 10):", clamped);
        UtilityFunctions::print("lerp(0, 100, 0.75):", lerped);
        UtilityFunctions::print("remap(5, 0-10, 100-200):", mapped);
    }

    void demonstrate_vector_math() {
        // Vector operations
        Vector2 a(3.0f, 4.0f);
        Vector2 b(1.0f, 2.0f);

        Vector2 sum = a + b;
        Vector2 diff = a - b;
        Vector2 scaled = a * 2.0f;
        float dot = a.dot(b);
        float length = a.length();
        Vector2 normalized = a.normalized();

        UtilityFunctions::print("Vector A:", a);
        UtilityFunctions::print("Vector B:", b);
        UtilityFunctions::print("A + B:", sum);
        UtilityFunctions::print("A - B:", diff);
        UtilityFunctions::print("A * 2:", scaled);
        UtilityFunctions::print("A dot B:", dot);
        UtilityFunctions::print("Length of A:", length);
        UtilityFunctions::print("A normalized:", normalized);

        // 3D vectors
        Vector3 v3a(1.0f, 2.0f, 3.0f);
        Vector3 v3b(4.0f, 5.0f, 6.0f);
        Vector3 cross = v3a.cross(v3b);

        UtilityFunctions::print("Cross product:", cross);
    }
};
```

### Type Conversion Utilities

```cpp
class TypeConversionUtilities : public RefCounted {
    GDCLASS(TypeConversionUtilities, RefCounted)

public:
    void demonstrate_type_conversions() {
        // Variant type checking
        Variant var_int = 42;
        Variant var_string = "Hello";
        Variant var_vector = Vector2(1.0f, 2.0f);

        UtilityFunctions::print("var_int type:", UtilityFunctions::typeof(var_int));
        UtilityFunctions::print("var_string type:", UtilityFunctions::typeof(var_string));
        UtilityFunctions::print("var_vector type:", UtilityFunctions::typeof(var_vector));

        // Type conversion
        String str_from_int = UtilityFunctions::str(var_int);  // "42"
        String str_from_vector = UtilityFunctions::str(var_vector);  // "(1, 2)"

        UtilityFunctions::print("String from int:", str_from_int);
        UtilityFunctions::print("String from vector:", str_from_vector);

        // Variant conversion
        if (var_int.get_type() == Variant::INT) {
            int value = var_int;
            UtilityFunctions::print("Converted to int:", value);
        }

        // Array/Dictionary utilities
        Array array = Array::make(1, 2, 3, "four", 5.0);
        Dictionary dict;
        dict["key1"] = "value1";
        dict["key2"] = 42;

        UtilityFunctions::print("Array:", array);
        UtilityFunctions::print("Dictionary:", dict);

        // JSON conversion
        JSON json;
        String json_string = json.stringify(dict);
        UtilityFunctions::print("JSON:", json_string);
    }

    void demonstrate_packed_arrays() {
        // Packed array conversions
        PackedFloat32Array floats;
        floats.append(1.1f);
        floats.append(2.2f);
        floats.append(3.3f);

        PackedInt32Array ints;
        for (int i = 0; i < floats.size(); i++) {
            ints.append(static_cast<int>(floats[i]));
        }

        PackedStringArray strings;
        for (int i = 0; i < ints.size(); i++) {
            strings.append(UtilityFunctions::str(ints[i]));
        }

        UtilityFunctions::print("Floats:", floats);
        UtilityFunctions::print("Ints:", ints);
        UtilityFunctions::print("Strings:", strings);

        // Byte array operations
        PackedByteArray bytes;
        String text = "Hello World";
        bytes = text.to_utf8_buffer();

        String decoded = String::utf8((const char*)bytes.ptr(), bytes.size());
        UtilityFunctions::print("Original:", text);
        UtilityFunctions::print("Decoded:", decoded);
    }
};
```

## Helper Macros

### Error Handling Macros

```cpp
class ErrorHandlingExamples : public RefCounted {
    GDCLASS(ErrorHandlingExamples, RefCounted)

public:
    bool validate_input(const String& input) {
        // Early return if condition fails
        ERR_FAIL_COND_V(input.is_empty(), false);
        ERR_FAIL_COND_V(input.length() > 255, false);

        // Continue with validation
        return true;
    }

    void process_array(const Array& arr) {
        // Fail and return if condition is true
        ERR_FAIL_COND(arr.is_empty());
        ERR_FAIL_INDEX(0, arr.size());

        // Process array...
        for (int i = 0; i < arr.size(); i++) {
            ERR_CONTINUE(arr[i].get_type() == Variant::NIL);  // Skip invalid entries
            UtilityFunctions::print("Item:", arr[i]);
        }
    }

    Node* get_child_safe(Node* parent, int index) {
        ERR_FAIL_NULL_V(parent, nullptr);
        ERR_FAIL_INDEX_V(index, parent->get_child_count(), nullptr);

        return parent->get_child(index);
    }

    void demonstrate_error_messages() {
        String filename = "";

        // Error with custom message
        ERR_FAIL_COND_MSG(filename.is_empty(), "Filename cannot be empty");

        // Print error without failing
        ERR_PRINT("This is an error message");
        WARN_PRINT("This is a warning message");

        // Print once (won't repeat)
        for (int i = 0; i < 5; i++) {
            ERR_PRINT_ONCE("This error appears only once");
        }
    }
};
```

### Memory Management Macros

```cpp
class MemoryManagementExamples : public RefCounted {
    GDCLASS(MemoryManagementExamples, RefCounted)

public:
    void demonstrate_memory_macros() {
        // Godot memory allocation
        Node* node = memnew(Node);
        node->set_name("DynamicNode");

        // Use the node...
        UtilityFunctions::print("Created node:", node->get_name());

        // Clean up
        memdelete(node);

        // Array allocation
        int* numbers = memnew_arr(int, 100);
        for (int i = 0; i < 100; i++) {
            numbers[i] = i * i;
        }

        // Use array...
        UtilityFunctions::print("Array[10]:", numbers[10]);

        // Clean up array
        memdelete_arr(numbers);
    }

    void demonstrate_reference_counting() {
        // Reference-counted objects
        Ref<Resource> resource = memnew(Resource);

        // Reference count is automatically managed
        Ref<Resource> another_ref = resource;
        UtilityFunctions::print("Reference count:", resource->get_reference_count());

        // When refs go out of scope, object is automatically deleted
    }

    Ref<Texture2D> load_texture_safe(const String& path) {
        // Safe resource loading
        Ref<Texture2D> texture = ResourceLoader::load(path);

        if (texture.is_null()) {
            ERR_PRINT("Failed to load texture: " + path);
            return Ref<Texture2D>();  // Return null ref
        }

        return texture;
    }
};
```

### Type Checking and Casting Macros

```cpp
class TypeCheckingExamples : public RefCounted {
    GDCLASS(TypeCheckingExamples, RefCounted)

public:
    void process_node(Node* node) {
        ERR_FAIL_NULL(node);

        // Safe casting
        if (Node2D* node2d = Object::cast_to<Node2D>(node)) {
            UtilityFunctions::print("Node2D position:", node2d->get_position());
        } else if (Node3D* node3d = Object::cast_to<Node3D>(node)) {
            UtilityFunctions::print("Node3D position:", node3d->get_position());
        } else if (Control* control = Object::cast_to<Control>(node)) {
            UtilityFunctions::print("Control size:", control->get_size());
        }

        // Class checking
        if (node->is_class("CharacterBody2D")) {
            UtilityFunctions::print("Found CharacterBody2D");
        }

        // Get class name
        StringName class_name = node->get_class();
        UtilityFunctions::print("Node class:", String(class_name));
    }

    void process_variant(const Variant& var) {
        // Type checking
        switch (var.get_type()) {
            case Variant::NIL:
                UtilityFunctions::print("Value is null");
                break;
            case Variant::BOOL:
                UtilityFunctions::print("Boolean value:", var.operator bool());
                break;
            case Variant::INT:
                UtilityFunctions::print("Integer value:", var.operator int());
                break;
            case Variant::FLOAT:
                UtilityFunctions::print("Float value:", var.operator double());
                break;
            case Variant::STRING:
                UtilityFunctions::print("String value:", var.operator String());
                break;
            case Variant::VECTOR2:
                UtilityFunctions::print("Vector2 value:", var.operator Vector2());
                break;
            case Variant::OBJECT:
                {
                    Object* obj = var;
                    if (obj) {
                        UtilityFunctions::print("Object class:", obj->get_class());
                    } else {
                        UtilityFunctions::print("Null object");
                    }
                }
                break;
            default:
                UtilityFunctions::print("Other type:", var.get_type());
                break;
        }
    }
};
```

## Random Number Generation

```cpp
class RandomUtilities : public RefCounted {
    GDCLASS(RandomUtilities, RefCounted)

private:
    Ref<RandomNumberGenerator> rng;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("generate_examples"),
                           &RandomUtilities::generate_examples);
    }

public:
    RandomUtilities() {
        rng.instantiate();
        rng->randomize();  // Seed with current time
    }

    void generate_examples() {
        // Basic random functions
        UtilityFunctions::print("Random float [0,1):", UtilityFunctions::randf());
        UtilityFunctions::print("Random int:", UtilityFunctions::randi());
        UtilityFunctions::print("Random range [5,15]:", UtilityFunctions::randi_range(5, 15));
        UtilityFunctions::print("Random float [-5,5]:", UtilityFunctions::randf_range(-5.0, 5.0));

        // Using RNG instance for reproducible results
        rng->set_seed(12345);  // Set specific seed

        UtilityFunctions::print("RNG float:", rng->randf());
        UtilityFunctions::print("RNG range [1,100]:", rng->randi_range(1, 100));

        // Random choice from array
        Array choices = Array::make("red", "green", "blue", "yellow");
        int random_index = rng->randi_range(0, choices.size() - 1);
        UtilityFunctions::print("Random choice:", choices[random_index]);

        // Random boolean
        bool coin_flip = rng->randf() < 0.5;
        UtilityFunctions::print("Coin flip:", coin_flip ? "heads" : "tails");

        // Weighted random choice
        float random_value = rng->randf();
        String rarity;
        if (random_value < 0.6) {
            rarity = "common";
        } else if (random_value < 0.85) {
            rarity = "uncommon";
        } else if (random_value < 0.95) {
            rarity = "rare";
        } else {
            rarity = "legendary";
        }
        UtilityFunctions::print("Item rarity:", rarity);
    }

    Vector2 random_point_in_circle(float radius) {
        float angle = rng->randf() * 2.0f * Math_PI;
        float distance = rng->randf() * radius;
        return Vector2(cos(angle) * distance, sin(angle) * distance);
    }

    Color random_color() {
        return Color(rng->randf(), rng->randf(), rng->randf(), 1.0f);
    }
};
```

## File and Path Utilities

```cpp
class FilePathUtilities : public RefCounted {
    GDCLASS(FilePathUtilities, RefCounted)

public:
    void demonstrate_path_functions() {
        String project_path = "res://scenes/main/player.tscn";
        String user_path = "user://save_data.json";
        String system_path = "/absolute/path/to/file.txt";

        // Path manipulation
        UtilityFunctions::print("Base dir:", project_path.get_base_dir());
        UtilityFunctions::print("File name:", project_path.get_file());
        UtilityFunctions::print("Extension:", project_path.get_extension());
        UtilityFunctions::print("Basename:", project_path.get_basename());

        // Path validation
        UtilityFunctions::print("Is absolute path:", system_path.is_absolute_path());
        UtilityFunctions::print("Is relative path:", project_path.is_relative_path());

        // Path operations
        String combined = "res://assets".path_join("textures").path_join("player.png");
        UtilityFunctions::print("Combined path:", combined);

        // File existence checking
        if (FileAccess::file_exists(project_path)) {
            UtilityFunctions::print("File exists:", project_path);
        }

        // Directory operations
        if (DirAccess::dir_exists_absolute("res://scenes")) {
            UtilityFunctions::print("Directory exists: res://scenes");
        }
    }

    void demonstrate_file_operations() {
        String file_path = "user://example.txt";
        String content = "Hello, World!\nThis is a test file.";

        // Write file
        Ref<FileAccess> file = FileAccess::open(file_path, FileAccess::WRITE);
        if (file.is_valid()) {
            file->store_string(content);
            file->close();
            UtilityFunctions::print("File written successfully");
        }

        // Read file
        file = FileAccess::open(file_path, FileAccess::READ);
        if (file.is_valid()) {
            String read_content = file->get_as_text();
            file->close();
            UtilityFunctions::print("File content:", read_content);
        }

        // Get file info
        uint64_t file_size = FileAccess::get_file_as_bytes(file_path).size();
        uint64_t modified_time = FileAccess::get_modified_time(file_path);

        UtilityFunctions::print("File size:", static_cast<int>(file_size));
        UtilityFunctions::print("Modified time:", static_cast<int>(modified_time));
    }
};
```

## Performance Utilities

```cpp
class PerformanceUtilities : public RefCounted {
    GDCLASS(PerformanceUtilities, RefCounted)

public:
    void demonstrate_timing() {
        uint64_t start_time = OS::get_singleton()->get_ticks_usec();

        // Do some work
        for (int i = 0; i < 100000; i++) {
            Math::sqrt(i);
        }

        uint64_t end_time = OS::get_singleton()->get_ticks_usec();
        uint64_t elapsed = end_time - start_time;

        UtilityFunctions::print("Operation took:", static_cast<int>(elapsed), "microseconds");
    }

    void demonstrate_memory_info() {
        // Memory usage information
        uint64_t static_mem = OS::get_singleton()->get_static_memory_usage();
        uint64_t max_static = OS::get_singleton()->get_static_memory_peak_usage();

        UtilityFunctions::print("Static memory usage:", static_cast<int>(static_mem));
        UtilityFunctions::print("Peak static memory:", static_cast<int>(max_static));
    }

    void demonstrate_profiling() {
        // Simple profiling
        const int iterations = 1000000;

        uint64_t start = OS::get_singleton()->get_ticks_usec();
        for (int i = 0; i < iterations; i++) {
            // Empty loop for baseline
        }
        uint64_t baseline = OS::get_singleton()->get_ticks_usec() - start;

        start = OS::get_singleton()->get_ticks_usec();
        for (int i = 0; i < iterations; i++) {
            Math::sin(i * 0.001f);
        }
        uint64_t sin_time = OS::get_singleton()->get_ticks_usec() - start;

        UtilityFunctions::print("Baseline time:", static_cast<int>(baseline));
        UtilityFunctions::print("Sin calculation time:", static_cast<int>(sin_time));
        UtilityFunctions::print("Sin overhead:", static_cast<int>(sin_time - baseline));
    }
};
```

## Conclusion

The utility functions and helper macros in godot-cpp provide essential tools for:

- **Debug output and logging** with `UtilityFunctions::print()` and error macros
- **String manipulation** with formatting, conversion, and validation
- **Mathematical operations** including trigonometry, interpolation, and vector math
- **Type conversion and checking** between C++ types and Variants
- **Memory management** with safe allocation and reference counting
- **File and path operations** for resource loading and data persistence
- **Random number generation** for procedural content and game mechanics
- **Performance measurement** with timing and profiling utilities

These utilities form the foundation for productive GDExtension development, providing familiar functionality with optimal performance and safety features.
