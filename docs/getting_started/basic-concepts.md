# Basic Concepts - Understanding GDExtension

## Introduction

**Essential concepts for GDExtension development:** This guide explains the fundamental concepts you need to understand when working with godot-cpp. These concepts form the foundation for everything from simple extensions to complex native libraries. Understanding these principles will help you design better extensions and avoid common pitfalls.

## What is GDExtension?

**A native extension system for Godot:** GDExtension allows you to write high-performance code in C++ that integrates seamlessly with Godot projects. Unlike GDNative (the predecessor), GDExtension provides better performance, improved memory management, and tighter integration with the engine.

### Key Benefits

- **Performance**: Native C++ speed for computationally intensive tasks
- **Library Integration**: Easy integration with existing C++ libraries
- **Type Safety**: Strong typing while maintaining GDScript compatibility
- **Hot Reloading**: Extensions can be reloaded during development
- **Cross-Platform**: Same code works on all Godot-supported platforms

## Core Concepts

### 1. Binary Interface Boundary

**The bridge between C++ and Godot:** GDExtension uses a stable binary interface (ABI) to communicate between your C++ code and the Godot engine. This means your extension can work with different versions of Godot without recompilation (within compatibility ranges).

```cpp
// Your C++ code
class MyClass : public Node {
    void my_method() {
        // This gets translated through the binary interface
        get_child(0)->queue_free();
    }
};

// Becomes a series of C function calls:
// 1. object_get_child(this, 0)
// 2. object_queue_free(child)
```

### 2. Object System Integration

**Your classes become Godot objects:** When you create a GDExtension class, it becomes a first-class Godot object. This means it has all the features of built-in Godot classes:

- **Unique ID**: Every object instance has a unique identifier
- **Signals**: Can emit and connect to signals
- **Properties**: Appear in the Inspector with proper serialization
- **Methods**: Can be called from GDScript and other languages
- **Inheritance**: Full inheritance chain is preserved

```cpp
class GameEntity : public Node2D {
    GDCLASS(GameEntity, Node2D)
    // Now GameEntity IS a Node2D with all its capabilities
};
```

### 3. Class Registration

**Making your classes visible to Godot:** Classes must be explicitly registered to be available in Godot. Different registration types provide different capabilities:

```cpp
// Standard class - can be instantiated
GDREGISTER_CLASS(MyNode);

// Virtual class - base class only, cannot instantiate
GDREGISTER_VIRTUAL_CLASS(BaseClass);

// Abstract class - has pure virtual methods
GDREGISTER_ABSTRACT_CLASS(AbstractBase);
```

### 4. Method Binding

**Exposing C++ methods to scripts:** Methods must be bound to be callable from GDScript. The binding system handles type conversion automatically:

```cpp
class Calculator : public RefCounted {
    GDCLASS(Calculator, RefCounted)

protected:
    static void _bind_methods() {
        // Bind method with argument names
        ClassDB::bind_method(D_METHOD("add", "a", "b"), &Calculator::add);
    }

public:
    int add(int a, int b) {
        return a + b;
    }
};

// In GDScript:
// var calc = Calculator.new()
// var result = calc.add(5, 3)  # Returns 8
```

### 5. Property System

**Exposing data with Inspector integration:** Properties automatically appear in the Inspector and can be saved/loaded:

```cpp
class Player : public CharacterBody2D {
private:
    float health = 100.0f;
    int level = 1;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("set_health", "health"), &Player::set_health);
        ClassDB::bind_method(D_METHOD("get_health"), &Player::get_health);
        
        // Property with range hint for Inspector
        ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "health", 
                                 PROPERTY_HINT_RANGE, "0,100,1"), 
                    "set_health", "get_health");
    }

public:
    void set_health(float p_health) { health = CLAMP(p_health, 0.0f, 100.0f); }
    float get_health() const { return health; }
};
```

### 6. Signal System

**Event-driven communication:** Signals provide loose coupling between objects and are the preferred way to communicate events:

```cpp
class EventEmitter : public Node {
    GDCLASS(EventEmitter, Node)

protected:
    static void _bind_methods() {
        // Define signals with their parameters
        ADD_SIGNAL(MethodInfo("health_changed", 
                             PropertyInfo(Variant::FLOAT, "new_health")));
        ADD_SIGNAL(MethodInfo("player_died"));
    }

public:
    void take_damage(float damage) {
        health -= damage;
        emit_signal("health_changed", health);
        
        if (health <= 0) {
            emit_signal("player_died");
        }
    }
};
```

### 7. Variant System

**Universal type for cross-language communication:** Variant is Godot's universal type that can hold any value. It handles automatic conversion between C++ types and script types:

```cpp
void process_data(const Variant &input) {
    // Variant can hold any type
    switch (input.get_type()) {
        case Variant::INT:
            int value = input;  // Automatic conversion
            break;
        case Variant::STRING:
            String text = input;
            break;
        case Variant::ARRAY:
            Array arr = input;
            for (int i = 0; i < arr.size(); i++) {
                process_data(arr[i]);  // Recursive processing
            }
            break;
    }
}
```

### 8. Memory Management

**Automatic and manual memory patterns:** GDExtension uses different memory management patterns depending on the class type:

```cpp
// Reference-counted objects (automatic cleanup)
class MyResource : public Resource {
    // Ref<MyResource> handles reference counting
};

// Manual memory management for Nodes
class MyNode : public Node {
    // Added to scene tree, cleaned up automatically
    // Or use queue_free() for deferred cleanup
};

// Stack allocation for data types
Vector3 position;  // Automatic cleanup when out of scope
String name;       // Automatic cleanup
```

## Inheritance and Object Hierarchy

### Godot's Object Hierarchy

Understanding Godot's object hierarchy helps you choose the right base class:

```
Object (Base of everything)
├── RefCounted (Reference-counted objects)
│   ├── Resource (Saveable data)
│   │   ├── Texture2D
│   │   ├── AudioStream
│   │   └── Your custom resources
│   └── Other ref-counted classes
└── Node (Scene tree objects)
    ├── Node2D (2D positioning)
    │   ├── Sprite2D
    │   ├── CharacterBody2D
    │   └── Your 2D nodes
    ├── Node3D (3D positioning)
    │   ├── MeshInstance3D
    │   ├── CharacterBody3D
    │   └── Your 3D nodes
    └── Control (UI elements)
        ├── Button
        ├── Label
        └── Your UI controls
```

### Choosing Base Classes

**Resource vs Node vs Object:**

```cpp
// Use Resource for data that can be saved/loaded
class GameData : public Resource {
    GDCLASS(GameData, Resource)
    // Can be saved as .tres files
    // Automatically reference-counted
};

// Use Node for scene tree objects
class GameManager : public Node {
    GDCLASS(GameManager, Node)
    // Can be added to scene tree
    // Has lifecycle methods (_ready, _process, etc.)
};

// Use RefCounted for utility classes
class Calculator : public RefCounted {
    GDCLASS(Calculator, RefCounted)
    // Automatic memory management
    // Not part of scene tree
};
```

## Lifecycle Methods

**Understanding when methods are called:** Godot calls specific methods at different points in an object's lifetime:

```cpp
class LifecycleExample : public Node {
    GDCLASS(LifecycleExample, Node)

public:
    // Constructor - called when object is created
    LifecycleExample() {
        print_line("Object constructed");
    }
    
    // Called when node enters the scene tree
    void _enter_tree() override {
        print_line("Entered tree");
    }
    
    // Called when node is ready (after children are ready)
    void _ready() override {
        print_line("Node ready");
    }
    
    // Called every frame
    void _process(double delta) override {
        // Frame-based logic
    }
    
    // Called every physics frame (fixed timestep)
    void _physics_process(double delta) override {
        // Physics logic
    }
    
    // Called when node exits the scene tree
    void _exit_tree() override {
        print_line("Exited tree");
    }
    
    // Destructor - called when object is destroyed
    ~LifecycleExample() {
        print_line("Object destroyed");
    }
};
```

## Type Conversion and Marshalling

**Automatic conversion between types:** The binding system automatically converts between C++ and script types:

```cpp
class TypeConverter : public RefCounted {
    GDCLASS(TypeConverter, RefCounted)

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("process_array", "data"), 
                           &TypeConverter::process_array);
        ClassDB::bind_method(D_METHOD("get_info"), 
                           &TypeConverter::get_info);
    }

public:
    // Array automatically converts between C++ and GDScript
    Array process_array(const Array &input) {
        Array result;
        for (int i = 0; i < input.size(); i++) {
            // Each element is automatically converted
            if (input[i].get_type() == Variant::INT) {
                int value = input[i];
                result.append(value * 2);
            }
        }
        return result;
    }
    
    // Dictionary automatically converts
    Dictionary get_info() {
        Dictionary info;
        info["version"] = "1.0";
        info["features"] = Array::make("fast", "reliable");
        return info;
    }
};
```

## Error Handling

**Robust error handling patterns:** GDExtension provides several error handling mechanisms:

```cpp
class ErrorHandling : public RefCounted {
    GDCLASS(ErrorHandling, RefCounted)

public:
    bool process_file(const String &path) {
        // Check preconditions
        ERR_FAIL_COND_V_MSG(path.is_empty(), false, "Path cannot be empty");
        
        // Use fail conditions
        FileAccess *file = FileAccess::open(path, FileAccess::READ);
        ERR_FAIL_NULL_V_MSG(file, false, "Cannot open file: " + path);
        
        // Process file...
        String content = file->get_as_text();
        file->close();
        
        // Validate content
        ERR_FAIL_COND_V_MSG(content.length() == 0, false, "File is empty");
        
        return true;
    }
    
    void safe_operation() {
        // Print warnings for non-critical issues
        WARN_PRINT("This is a warning message");
        
        // Print errors for critical issues
        ERR_PRINT("This is an error message");
        
        // Conditional warnings and errors
        WARN_PRINT_ONCE("This warning appears only once");
        ERR_PRINT_ONCE("This error appears only once");
    }
};
```

## Best Practices

### 1. Design Principles

```cpp
// Good: Single responsibility
class AudioManager : public Node {
    // Only handles audio-related functionality
};

// Good: Clear inheritance
class CustomButton : public Button {
    // Extends existing functionality
};

// Avoid: Overly complex classes
class EverythingManager : public Node {
    // Handles audio, input, networking, AI... (too much)
};
```

### 2. Naming Conventions

```cpp
class MyCustomNode : public Node2D {  // PascalCase for classes
    GDCLASS(MyCustomNode, Node2D)

private:
    int player_health;          // snake_case for variables
    float movement_speed;

protected:
    static void _bind_methods() {
        // snake_case for GDScript-visible methods
        ClassDB::bind_method(D_METHOD("get_player_health"), 
                           &MyCustomNode::get_player_health);
    }

public:
    int get_player_health() const { return player_health; }  // C++ style
    void set_player_health(int p_health) { player_health = p_health; }
};
```

### 3. Resource Management

```cpp
class ResourceManager : public Node {
    GDCLASS(ResourceManager, Node)

private:
    Vector<Ref<Texture2D>> textures;  // Use Ref<> for reference-counted
    FileAccess *file = nullptr;       // Raw pointer needs manual cleanup

public:
    ~ResourceManager() {
        // Clean up raw pointers
        if (file) {
            file->close();
            memdelete(file);
        }
        // Ref<> objects clean themselves up
    }
};
```

## Common Patterns

### Singleton Pattern

```cpp
class GameSettings : public RefCounted {
    GDCLASS(GameSettings, RefCounted)

private:
    static GameSettings *instance;

protected:
    static void _bind_methods() {
        ClassDB::bind_static_method("GameSettings", 
                                   D_METHOD("get_instance"), 
                                   &GameSettings::get_instance);
    }

public:
    static GameSettings *get_instance() {
        if (!instance) {
            instance = memnew(GameSettings);
        }
        return instance;
    }
};

// In GDScript: var settings = GameSettings.get_instance()
```

### Factory Pattern

```cpp
class EntityFactory : public RefCounted {
    GDCLASS(EntityFactory, RefCounted)

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("create_entity", "type"), 
                           &EntityFactory::create_entity);
    }

public:
    Ref<RefCounted> create_entity(const String &type) {
        if (type == "player") {
            return memnew(Player);
        } else if (type == "enemy") {
            return memnew(Enemy);
        }
        return nullptr;
    }
};
```

## Common Mistakes to Avoid

### 1. Memory Leaks

```cpp
// Bad: Not cleaning up raw pointers
class BadExample : public Node {
    Object *obj = memnew(Object);  // Never deleted!
};

// Good: Use Ref<> or proper cleanup
class GoodExample : public Node {
    Ref<RefCounted> obj;  // Automatically cleaned up
    
    ~GoodExample() {
        if (raw_pointer) {
            memdelete(raw_pointer);
        }
    }
};
```

### 2. Forgetting Registration

```cpp
// The class exists but isn't registered
class MyClass : public Node {
    GDCLASS(MyClass, Node)
};

// Must register in initialization
void initialize_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }
    GDREGISTER_CLASS(MyClass);  // Don't forget this!
}
```

### 3. Incorrect Binding

```cpp
// Bad: Method not bound
class Example : public RefCounted {
    GDCLASS(Example, RefCounted)

public:
    int calculate(int x) { return x * 2; }  // Not accessible from GDScript
};

// Good: Proper binding
class Example : public RefCounted {
    GDCLASS(Example, RefCounted)

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("calculate", "x"), &Example::calculate);
    }
};
```

## Next Steps

Now that you understand the basic concepts:

1. **Build Practice Projects**: Create simple extensions to reinforce concepts
2. **Study Architecture**: Learn about [Core Architecture](../architecture/core-architecture.md)
3. **Explore Integration**: Understand [Engine Integration](../engine-integration/)
4. **Advanced Topics**: Move on to [Advanced Topics](../advanced-topics/) when ready

## Conclusion

These fundamental concepts form the foundation of all GDExtension development:

- **Classes** integrate fully with Godot's object system
- **Registration** makes classes available to scripts
- **Binding** exposes methods and properties
- **Signals** provide event communication
- **Memory management** follows clear patterns
- **Type conversion** happens automatically

Understanding these concepts will help you build robust, efficient, and maintainable GDExtensions that feel natural to both C++ developers and GDScript users.