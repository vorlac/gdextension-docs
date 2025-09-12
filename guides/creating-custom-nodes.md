# Creating Custom Nodes

## Introduction

**Building custom nodes that integrate seamlessly with Godot:** Custom nodes are the most common type of GDExtension, extending Godot's node system with specialized functionality. This guide covers everything from simple utility nodes to complex systems that integrate with the editor and provide rich Inspector interfaces.

## Choosing the Right Base Class

**Selecting the appropriate parent class:** The base class you inherit from determines your node's capabilities and where it appears in the editor's node creation dialog.

### Node Hierarchy Overview

```cpp
// Base classes and their use cases:

// Object - Base for everything, minimal overhead
class DataProcessor : public Object {
    // Use for: Utilities, managers, data structures
};

// RefCounted - Automatic memory management
class ConfigManager : public RefCounted {
    // Use for: Shared objects, cached data, utilities
};

// Node - Basic scene tree functionality
class GameManager : public Node {
    // Use for: Managers, controllers, invisible logic nodes
};

// Node2D - 2D positioning and transforms
class CustomSprite : public Node2D {
    // Use for: 2D game objects, visual effects, UI overlays
};

// Node3D - 3D positioning and transforms
class CustomMesh : public Node3D {
    // Use for: 3D game objects, spatial effects, 3D utilities
};

// Control - UI and layout system
class CustomButton : public Control {
    // Use for: UI widgets, HUD elements, menus
};

// Resource - Saveable data
class GameSettings : public Resource {
    // Use for: Configuration data, game assets, persistent data
};
```

### Decision Matrix

| Base Class | Scene Tree | Transform | UI Layout | Auto-Save | Memory | Use Cases |
|------------|------------|-----------|-----------|-----------|---------|-----------|
| **Object** | ❌ | ❌ | ❌ | ❌ | Manual | Data processors, utilities |
| **RefCounted** | ❌ | ❌ | ❌ | ❌ | Auto | Shared objects, managers |
| **Node** | ✅ | ❌ | ❌ | ❌ | Scene | Logic controllers, managers |
| **Node2D** | ✅ | 2D | ❌ | ❌ | Scene | 2D game objects, effects |
| **Node3D** | ✅ | 3D | ❌ | ❌ | Scene | 3D game objects, spatial |
| **Control** | ✅ | 2D | ✅ | ❌ | Scene | UI widgets, HUD elements |
| **Resource** | ❌ | ❌ | ❌ | ✅ | Auto | Assets, settings, data |

## Creating a Basic Custom Node

### Step 1: Header File Structure

```cpp
// custom_timer.h
#ifndef CUSTOM_TIMER_H
#define CUSTOM_TIMER_H

#include <godot_cpp/classes/node.hpp>
#include <godot_cpp/core/class_db.hpp>

using namespace godot;

class CustomTimer : public Node {
    GDCLASS(CustomTimer, Node)

private:
    double time_left;
    double wait_time;
    bool autostart;
    bool paused;
    bool one_shot;

protected:
    static void _bind_methods();
    void _notification(int p_what);

public:
    CustomTimer();
    ~CustomTimer();

    // Core functionality
    void start(double p_time = -1);
    void stop();
    void set_paused(bool p_paused);
    bool is_stopped() const;
    double get_time_left() const;

    // Properties
    void set_wait_time(double p_time);
    double get_wait_time() const;

    void set_autostart(bool p_enabled);
    bool has_autostart() const;

    void set_one_shot(bool p_enabled);
    bool is_one_shot() const;

    // Virtual overrides
    void _ready() override;
    void _process(double delta) override;
};

#endif // CUSTOM_TIMER_H
```

### Step 2: Implementation File

```cpp
// custom_timer.cpp
#include "custom_timer.h"
#include <godot_cpp/variant/utility_functions.hpp>

void CustomTimer::_bind_methods() {
    // Bind methods for GDScript access
    ClassDB::bind_method(D_METHOD("start", "time"), &CustomTimer::start, DEFVAL(-1));
    ClassDB::bind_method(D_METHOD("stop"), &CustomTimer::stop);
    ClassDB::bind_method(D_METHOD("set_paused", "paused"), &CustomTimer::set_paused);
    ClassDB::bind_method(D_METHOD("is_stopped"), &CustomTimer::is_stopped);
    ClassDB::bind_method(D_METHOD("get_time_left"), &CustomTimer::get_time_left);

    // Bind properties with getters/setters
    ClassDB::bind_method(D_METHOD("set_wait_time", "time"), &CustomTimer::set_wait_time);
    ClassDB::bind_method(D_METHOD("get_wait_time"), &CustomTimer::get_wait_time);
    ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "wait_time", PROPERTY_HINT_RANGE, "0,3600,0.01,or_greater"),
                 "set_wait_time", "get_wait_time");

    ClassDB::bind_method(D_METHOD("set_autostart", "enabled"), &CustomTimer::set_autostart);
    ClassDB::bind_method(D_METHOD("has_autostart"), &CustomTimer::has_autostart);
    ADD_PROPERTY(PropertyInfo(Variant::BOOL, "autostart"),
                 "set_autostart", "has_autostart");

    ClassDB::bind_method(D_METHOD("set_one_shot", "enabled"), &CustomTimer::set_one_shot);
    ClassDB::bind_method(D_METHOD("is_one_shot"), &CustomTimer::is_one_shot);
    ADD_PROPERTY(PropertyInfo(Variant::BOOL, "one_shot"),
                 "set_one_shot", "is_one_shot");

    // Add property groups for organization
    ADD_GROUP("Timer", "");

    // Bind signals
    ADD_SIGNAL(MethodInfo("timeout"));
    ADD_SIGNAL(MethodInfo("started"));
    ADD_SIGNAL(MethodInfo("stopped"));
}

CustomTimer::CustomTimer() {
    time_left = 0.0;
    wait_time = 1.0;
    autostart = false;
    paused = false;
    one_shot = false;

    // Don't process by default
    set_process(false);
}

CustomTimer::~CustomTimer() {
    // Cleanup if needed
}

void CustomTimer::_notification(int p_what) {
    switch (p_what) {
        case NOTIFICATION_READY:
            if (autostart) {
                start();
            }
            break;
    }
}

void CustomTimer::_ready() {
    // Called when node is ready
    UtilityFunctions::print("CustomTimer ready");
}

void CustomTimer::_process(double delta) {
    if (paused || time_left <= 0) {
        return;
    }

    time_left -= delta;

    if (time_left <= 0) {
        time_left = 0;
        set_process(false);

        emit_signal("timeout");

        if (!one_shot) {
            // Restart for repeating timer
            time_left = wait_time;
            set_process(true);
        }
    }
}

void CustomTimer::start(double p_time) {
    if (p_time > 0) {
        wait_time = p_time;
    }

    time_left = wait_time;
    paused = false;
    set_process(true);

    emit_signal("started");
}

void CustomTimer::stop() {
    time_left = 0;
    set_process(false);
    emit_signal("stopped");
}

void CustomTimer::set_paused(bool p_paused) {
    if (paused != p_paused) {
        paused = p_paused;
        set_process(!paused && time_left > 0);
    }
}

bool CustomTimer::is_stopped() const {
    return time_left <= 0;
}

double CustomTimer::get_time_left() const {
    return MAX(0, time_left);
}

void CustomTimer::set_wait_time(double p_time) {
    wait_time = MAX(0, p_time);
}

double CustomTimer::get_wait_time() const {
    return wait_time;
}

void CustomTimer::set_autostart(bool p_enabled) {
    autostart = p_enabled;
}

bool CustomTimer::has_autostart() const {
    return autostart;
}

void CustomTimer::set_one_shot(bool p_enabled) {
    one_shot = p_enabled;
}

bool CustomTimer::is_one_shot() const {
    return one_shot;
}
```

## Advanced Custom Node Features

### Rich Inspector Properties

```cpp
class AdvancedCustomNode : public Node2D {
    GDCLASS(AdvancedCustomNode, Node2D)

private:
    // Various property types
    int health = 100;
    float speed = 150.0f;
    String character_name = "Hero";
    Color tint_color = Color(1, 1, 1, 1);
    Vector2 offset = Vector2(0, 0);

    // Enum property
    enum MovementType {
        MOVEMENT_WALK,
        MOVEMENT_RUN,
        MOVEMENT_FLY
    };
    MovementType movement_type = MOVEMENT_WALK;

    // Resource property
    Ref<Texture2D> icon_texture;

    // Array properties
    PackedStringArray tags;
    Array inventory_items;

    // Node path property
    NodePath target_path;

    // File path property
    String config_file_path = "res://config.cfg";

protected:
    static void _bind_methods() {
        // Basic properties
        ClassDB::bind_method(D_METHOD("set_health", "health"), &AdvancedCustomNode::set_health);
        ClassDB::bind_method(D_METHOD("get_health"), &AdvancedCustomNode::get_health);
        ADD_PROPERTY(PropertyInfo(Variant::INT, "health", PROPERTY_HINT_RANGE, "0,999,1"),
                     "set_health", "get_health");

        ClassDB::bind_method(D_METHOD("set_speed", "speed"), &AdvancedCustomNode::set_speed);
        ClassDB::bind_method(D_METHOD("get_speed"), &AdvancedCustomNode::get_speed);
        ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "speed", PROPERTY_HINT_RANGE, "0,1000,0.1"),
                     "set_speed", "get_speed");

        // String with length limit
        ClassDB::bind_method(D_METHOD("set_character_name", "name"), &AdvancedCustomNode::set_character_name);
        ClassDB::bind_method(D_METHOD("get_character_name"), &AdvancedCustomNode::get_character_name);
        ADD_PROPERTY(PropertyInfo(Variant::STRING, "character_name", PROPERTY_HINT_LENGTH, "1,32"),
                     "set_character_name", "get_character_name");

        // Color picker
        ClassDB::bind_method(D_METHOD("set_tint_color", "color"), &AdvancedCustomNode::set_tint_color);
        ClassDB::bind_method(D_METHOD("get_tint_color"), &AdvancedCustomNode::get_tint_color);
        ADD_PROPERTY(PropertyInfo(Variant::COLOR, "tint_color"),
                     "set_tint_color", "get_tint_color");

        // Vector with step
        ClassDB::bind_method(D_METHOD("set_offset", "offset"), &AdvancedCustomNode::set_offset);
        ClassDB::bind_method(D_METHOD("get_offset"), &AdvancedCustomNode::get_offset);
        ADD_PROPERTY(PropertyInfo(Variant::VECTOR2, "offset", PROPERTY_HINT_NONE, "", PROPERTY_USAGE_DEFAULT),
                     "set_offset", "get_offset");

        // Enum dropdown
        ClassDB::bind_method(D_METHOD("set_movement_type", "type"), &AdvancedCustomNode::set_movement_type);
        ClassDB::bind_method(D_METHOD("get_movement_type"), &AdvancedCustomNode::get_movement_type);
        ADD_PROPERTY(PropertyInfo(Variant::INT, "movement_type", PROPERTY_HINT_ENUM, "Walk,Run,Fly"),
                     "set_movement_type", "get_movement_type");

        // Resource property
        ClassDB::bind_method(D_METHOD("set_icon_texture", "texture"), &AdvancedCustomNode::set_icon_texture);
        ClassDB::bind_method(D_METHOD("get_icon_texture"), &AdvancedCustomNode::get_icon_texture);
        ADD_PROPERTY(PropertyInfo(Variant::OBJECT, "icon_texture", PROPERTY_HINT_RESOURCE_TYPE, "Texture2D"),
                     "set_icon_texture", "get_icon_texture");

        // Array properties
        ClassDB::bind_method(D_METHOD("set_tags", "tags"), &AdvancedCustomNode::set_tags);
        ClassDB::bind_method(D_METHOD("get_tags"), &AdvancedCustomNode::get_tags);
        ADD_PROPERTY(PropertyInfo(Variant::PACKED_STRING_ARRAY, "tags"),
                     "set_tags", "get_tags");

        // Node path
        ClassDB::bind_method(D_METHOD("set_target_path", "path"), &AdvancedCustomNode::set_target_path);
        ClassDB::bind_method(D_METHOD("get_target_path"), &AdvancedCustomNode::get_target_path);
        ADD_PROPERTY(PropertyInfo(Variant::NODE_PATH, "target_path"),
                     "set_target_path", "get_target_path");

        // File path
        ClassDB::bind_method(D_METHOD("set_config_file_path", "path"), &AdvancedCustomNode::set_config_file_path);
        ClassDB::bind_method(D_METHOD("get_config_file_path"), &AdvancedCustomNode::get_config_file_path);
        ADD_PROPERTY(PropertyInfo(Variant::STRING, "config_file_path", PROPERTY_HINT_FILE, "*.cfg,*.ini"),
                     "set_config_file_path", "get_config_file_path");

        // Property groups
        ADD_GROUP("Character", "character_");
        ADD_GROUP("Movement", "movement_");
        ADD_GROUP("Visual", "");
        ADD_GROUP("Paths", "");

        // Bind enum constants
        BIND_ENUM_CONSTANT(MOVEMENT_WALK);
        BIND_ENUM_CONSTANT(MOVEMENT_RUN);
        BIND_ENUM_CONSTANT(MOVEMENT_FLY);
    }

public:
    // Property implementations
    void set_health(int p_health) {
        health = CLAMP(p_health, 0, 999);
        notify_property_list_changed();  // Refresh inspector if needed
    }
    int get_health() const { return health; }

    void set_speed(float p_speed) { speed = MAX(0, p_speed); }
    float get_speed() const { return speed; }

    void set_character_name(const String& p_name) {
        character_name = p_name.substr(0, 32);  // Enforce length limit
    }
    String get_character_name() const { return character_name; }

    void set_tint_color(const Color& p_color) { tint_color = p_color; }
    Color get_tint_color() const { return tint_color; }

    void set_offset(const Vector2& p_offset) { offset = p_offset; }
    Vector2 get_offset() const { return offset; }

    void set_movement_type(int p_type) {
        movement_type = static_cast<MovementType>(CLAMP(p_type, 0, 2));
    }
    int get_movement_type() const { return movement_type; }

    void set_icon_texture(const Ref<Texture2D>& p_texture) { icon_texture = p_texture; }
    Ref<Texture2D> get_icon_texture() const { return icon_texture; }

    void set_tags(const PackedStringArray& p_tags) { tags = p_tags; }
    PackedStringArray get_tags() const { return tags; }

    void set_target_path(const NodePath& p_path) { target_path = p_path; }
    NodePath get_target_path() const { return target_path; }

    void set_config_file_path(const String& p_path) { config_file_path = p_path; }
    String get_config_file_path() const { return config_file_path; }
};

VARIANT_ENUM_CAST(AdvancedCustomNode::MovementType);
```

### Dynamic Properties

```cpp
class DynamicPropertyNode : public Node {
    GDCLASS(DynamicPropertyNode, Node)

private:
    Dictionary dynamic_properties;
    bool show_advanced = false;
    String property_filter = "";

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("set_show_advanced", "show"), &DynamicPropertyNode::set_show_advanced);
        ClassDB::bind_method(D_METHOD("get_show_advanced"), &DynamicPropertyNode::get_show_advanced);

        ClassDB::bind_method(D_METHOD("set_property_filter", "filter"), &DynamicPropertyNode::set_property_filter);
        ClassDB::bind_method(D_METHOD("get_property_filter"), &DynamicPropertyNode::get_property_filter);

        // This will appear in the inspector
        ADD_PROPERTY(PropertyInfo(Variant::BOOL, "show_advanced"),
                     "set_show_advanced", "get_show_advanced");
        ADD_PROPERTY(PropertyInfo(Variant::STRING, "property_filter", PROPERTY_HINT_PLACEHOLDER_TEXT, "Filter..."),
                     "set_property_filter", "get_property_filter");
    }

    // Override to provide dynamic property list
    virtual void _get_property_list(List<PropertyInfo>* p_list) const override {
        // Add dynamic properties based on current settings
        if (show_advanced) {
            p_list->push_back(PropertyInfo(Variant::FLOAT, "advanced_multiplier",
                                          PROPERTY_HINT_RANGE, "0,10,0.1"));
            p_list->push_back(PropertyInfo(Variant::VECTOR3, "advanced_offset"));
        }

        // Add filtered properties
        if (!property_filter.is_empty()) {
            // Add properties that match the filter
            p_list->push_back(PropertyInfo(Variant::STRING, "filtered_property_1"));
            p_list->push_back(PropertyInfo(Variant::STRING, "filtered_property_2"));
        }

        // Add separator
        p_list->push_back(PropertyInfo(Variant::NIL, "", PROPERTY_HINT_NONE, "", PROPERTY_USAGE_GROUP, "Dynamic Properties"));

        // Add properties from dictionary
        for (const KeyValue<Variant, Variant>& E : dynamic_properties) {
            String prop_name = "dynamic_" + String(E.key);
            Variant::Type type = E.value.get_type();
            p_list->push_back(PropertyInfo(type, prop_name));
        }
    }

    // Override to handle dynamic property getting
    virtual bool _get(const StringName& p_name, Variant& r_ret) const override {
        String name = p_name;

        if (name == "advanced_multiplier") {
            r_ret = dynamic_properties.get("advanced_multiplier", 1.0);
            return true;
        } else if (name == "advanced_offset") {
            r_ret = dynamic_properties.get("advanced_offset", Vector3());
            return true;
        } else if (name.begins_with("filtered_property_")) {
            r_ret = dynamic_properties.get(name, "");
            return true;
        } else if (name.begins_with("dynamic_")) {
            String key = name.substr(8);  // Remove "dynamic_" prefix
            r_ret = dynamic_properties.get(key, Variant());
            return true;
        }

        return false;
    }

    // Override to handle dynamic property setting
    virtual bool _set(const StringName& p_name, const Variant& p_value) override {
        String name = p_name;

        if (name == "advanced_multiplier" || name == "advanced_offset" ||
            name.begins_with("filtered_property_") || name.begins_with("dynamic_")) {
            String key = name.begins_with("dynamic_") ? name.substr(8) : name;
            dynamic_properties[key] = p_value;
            return true;
        }

        return false;
    }

public:
    void set_show_advanced(bool p_show) {
        if (show_advanced != p_show) {
            show_advanced = p_show;
            notify_property_list_changed();
        }
    }

    bool get_show_advanced() const {
        return show_advanced;
    }

    void set_property_filter(const String& p_filter) {
        if (property_filter != p_filter) {
            property_filter = p_filter;
            notify_property_list_changed();
        }
    }

    String get_property_filter() const {
        return property_filter;
    }

    void add_dynamic_property(const String& name, const Variant& value) {
        dynamic_properties[name] = value;
        notify_property_list_changed();
    }

    void remove_dynamic_property(const String& name) {
        dynamic_properties.erase(name);
        notify_property_list_changed();
    }
};
```

## Tool Scripts and Editor Integration

### Making Your Node Work in the Editor

```cpp
class EditorIntegratedNode : public Node2D {
    GDCLASS(EditorIntegratedNode, Node2D)

private:
    Vector<Vector2> debug_points;
    Color debug_color = Color(1, 0, 0, 0.5);
    bool show_debug = true;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("set_show_debug", "show"), &EditorIntegratedNode::set_show_debug);
        ClassDB::bind_method(D_METHOD("get_show_debug"), &EditorIntegratedNode::get_show_debug);
        ADD_PROPERTY(PropertyInfo(Variant::BOOL, "show_debug"),
                     "set_show_debug", "get_show_debug");

        ClassDB::bind_method(D_METHOD("set_debug_color", "color"), &EditorIntegratedNode::set_debug_color);
        ClassDB::bind_method(D_METHOD("get_debug_color"), &EditorIntegratedNode::get_debug_color);
        ADD_PROPERTY(PropertyInfo(Variant::COLOR, "debug_color"),
                     "set_debug_color", "get_debug_color");

        ClassDB::bind_method(D_METHOD("add_debug_point", "point"), &EditorIntegratedNode::add_debug_point);
        ClassDB::bind_method(D_METHOD("clear_debug_points"), &EditorIntegratedNode::clear_debug_points);
    }

public:
    void _ready() override {
        // Different behavior in editor vs game
        if (Engine::get_singleton()->is_editor_hint()) {
            // Running in editor - setup editor-specific behavior
            setup_editor_mode();
        } else {
            // Running in game - setup game behavior
            setup_game_mode();
        }
    }

    void _process(double delta) override {
        if (Engine::get_singleton()->is_editor_hint()) {
            // Editor processing
            update_editor_visuals();
        }

        // Always update, but queue_redraw for editor
        if (Engine::get_singleton()->is_editor_hint()) {
            queue_redraw();  // Refresh editor display
        }
    }

    void _draw() override {
        if (!show_debug) return;

        // Draw debug information
        for (int i = 0; i < debug_points.size(); i++) {
            draw_circle(debug_points[i], 5.0f, debug_color);

            if (i > 0) {
                draw_line(debug_points[i-1], debug_points[i], debug_color, 2.0f);
            }
        }
    }

    // Editor-specific functionality
    void set_show_debug(bool p_show) {
        show_debug = p_show;
        queue_redraw();
    }

    bool get_show_debug() const {
        return show_debug;
    }

    void set_debug_color(const Color& p_color) {
        debug_color = p_color;
        queue_redraw();
    }

    Color get_debug_color() const {
        return debug_color;
    }

    void add_debug_point(const Vector2& point) {
        debug_points.append(point);
        queue_redraw();
    }

    void clear_debug_points() {
        debug_points.clear();
        queue_redraw();
    }

private:
    void setup_editor_mode() {
        // Editor-only setup
        set_process(true);  // Enable processing in editor

        // Add some sample debug points
        debug_points.append(Vector2(0, 0));
        debug_points.append(Vector2(50, 50));
        debug_points.append(Vector2(100, 0));
    }

    void setup_game_mode() {
        // Game-only setup
        show_debug = false;  // Hide debug visuals in game
        debug_points.clear();
    }

    void update_editor_visuals() {
        // Update editor-specific visuals
        // This runs only in the editor
    }
};
```

### Custom Gizmos and Handles

```cpp
class GizmoNode : public Node2D {
    GDCLASS(GizmoNode, Node2D)

private:
    float radius = 50.0f;
    Vector2 handle_position = Vector2(50, 0);

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("set_radius", "radius"), &GizmoNode::set_radius);
        ClassDB::bind_method(D_METHOD("get_radius"), &GizmoNode::get_radius);
        ADD_PROPERTY(PropertyInfo(Variant::FLOAT, "radius", PROPERTY_HINT_RANGE, "1,500,1"),
                     "set_radius", "get_radius");
    }

public:
    void _draw() override {
        if (Engine::get_singleton()->is_editor_hint()) {
            // Draw circle outline
            draw_arc(Vector2(), radius, 0, Math_TAU, 64, Color(1, 1, 1, 0.5), 2.0f);

            // Draw handle
            draw_circle(handle_position, 5.0f, Color(1, 0, 0));
        }
    }

    void set_radius(float p_radius) {
        radius = MAX(1.0f, p_radius);
        handle_position = Vector2(radius, 0);
        queue_redraw();
    }

    float get_radius() const {
        return radius;
    }

    // Handle mouse interaction in editor
    virtual bool _can_drop_data(const Vector2& position, const Variant& data) const override {
        return Engine::get_singleton()->is_editor_hint();
    }

    virtual void _drop_data(const Vector2& position, const Variant& data) override {
        if (Engine::get_singleton()->is_editor_hint()) {
            // Handle dropped data in editor
            handle_position = position;
            radius = position.length();
            queue_redraw();
        }
    }
};
```

## Node Registration and Organization

### Registering Your Custom Nodes

```cpp
// In your module initialization file
void initialize_custom_nodes_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }

    // Register all your custom nodes
    GDREGISTER_CLASS(CustomTimer);
    GDREGISTER_CLASS(AdvancedCustomNode);
    GDREGISTER_CLASS(DynamicPropertyNode);
    GDREGISTER_CLASS(EditorIntegratedNode);
    GDREGISTER_CLASS(GizmoNode);

    // You can also register virtual base classes
    // GDREGISTER_VIRTUAL_CLASS(BaseCustomNode);
}

void uninitialize_custom_nodes_module(ModuleInitializationLevel p_level) {
    if (p_level != MODULE_INITIALIZATION_LEVEL_SCENE) {
        return;
    }

    // Cleanup if needed
}
```

### Node Categories and Icons

```cpp
// You can organize nodes in categories by naming conventions
class UI_CustomButton : public Control {
    GDCLASS(UI_CustomButton, Control)
    // Will appear under UI category in create dialog
};

class Game_InventoryManager : public Node {
    GDCLASS(Game_InventoryManager, Node)
    // Will appear under Game category
};

class Audio_SoundController : public Node {
    GDCLASS(Audio_SoundController, Node)
    // Will appear under Audio category
};
```

## Best Practices and Common Patterns

### 1. Proper Initialization Order

```cpp
class ProperlyInitializedNode : public Node2D {
    GDCLASS(ProperlyInitializedNode, Node2D)

private:
    Ref<Resource> required_resource;
    Node* cached_child = nullptr;

public:
    void _enter_tree() override {
        // Called when node enters scene tree
        // Good for: Initial setup that doesn't depend on children
    }

    void _ready() override {
        // Called when node and all children are ready
        // Good for: Finding child nodes, connecting signals

        cached_child = get_node_or_null(NodePath("ChildNode"));
        if (cached_child) {
            // Connect to child signals
            cached_child->connect("signal_name", Callable(this, "on_child_signal"));
        }

        // Load resources
        required_resource = preload("res://assets/data.tres");
    }

    void _exit_tree() override {
        // Called when node exits scene tree
        // Good for: Cleanup, disconnecting signals

        if (cached_child && cached_child->is_connected("signal_name", Callable(this, "on_child_signal"))) {
            cached_child->disconnect("signal_name", Callable(this, "on_child_signal"));
        }
    }
};
```

### 2. Signal-Based Communication

```cpp
class SignalBasedNode : public Node {
    GDCLASS(SignalBasedNode, Node)

protected:
    static void _bind_methods() {
        // Define clear, descriptive signals
        ADD_SIGNAL(MethodInfo("health_changed",
                             PropertyInfo(Variant::INT, "new_health"),
                             PropertyInfo(Variant::INT, "old_health")));

        ADD_SIGNAL(MethodInfo("item_collected",
                             PropertyInfo(Variant::STRING, "item_name"),
                             PropertyInfo(Variant::INT, "quantity")));

        ADD_SIGNAL(MethodInfo("state_changed",
                             PropertyInfo(Variant::STRING, "new_state"),
                             PropertyInfo(Variant::STRING, "previous_state")));
    }

public:
    void take_damage(int damage) {
        int old_health = health;
        health -= damage;

        // Always emit specific, detailed signals
        emit_signal("health_changed", health, old_health);

        if (health <= 0) {
            emit_signal("state_changed", "dead", "alive");
        }
    }

    void collect_item(const String& item_name, int quantity = 1) {
        // Process collection logic...

        emit_signal("item_collected", item_name, quantity);
    }

private:
    int health = 100;
};
```

### 3. Performance Optimization

```cpp
class OptimizedNode : public Node2D {
    GDCLASS(OptimizedNode, Node2D)

private:
    bool needs_update = true;
    float update_interval = 0.1f;
    float time_since_update = 0.0f;

public:
    void _ready() override {
        // Only enable processing when needed
        set_process(false);
        set_physics_process(false);

        // Start processing only when required
        start_processing();
    }

    void _process(double delta) override {
        // Throttle updates
        time_since_update += delta;
        if (time_since_update < update_interval) {
            return;
        }
        time_since_update = 0.0f;

        // Only update when needed
        if (needs_update) {
            perform_update();
            needs_update = false;

            // Disable processing when done
            if (!has_pending_updates()) {
                set_process(false);
            }
        }
    }

    void mark_dirty() {
        needs_update = true;
        if (!is_processing()) {
            set_process(true);
        }
    }

private:
    void start_processing() {
        set_process(true);
    }

    void perform_update() {
        // Actual update logic
    }

    bool has_pending_updates() {
        // Check if more updates are needed
        return needs_update;
    }
};
```

## Conclusion

Creating custom nodes involves:

1. **Choosing the right base class** based on functionality needs
2. **Proper method binding** for GDScript integration
3. **Rich property systems** with appropriate hints and validation
4. **Signal communication** for loose coupling
5. **Editor integration** for development workflow
6. **Performance optimization** through selective processing
7. **Proper registration** and organization

Well-designed custom nodes feel like native Godot classes, integrate seamlessly with the editor, and provide clear, useful functionality to game developers.
