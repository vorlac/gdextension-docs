# Resource Management - Godot Assets

## Introduction

**Mastering resource lifecycle in GDExtension:** Resource management is crucial for creating stable, performant GDExtensions. This guide covers memory management patterns, reference counting, asset loading, and cleanup strategies to prevent memory leaks and crashes.

## Understanding Godot's Memory Model

**Different memory management patterns for different object types:** Godot uses multiple memory management strategies depending on the object type, and GDExtension code must respect these patterns.

### Memory Management Categories

```cpp
class MemoryModelExample : public RefCounted {
    GDCLASS(MemoryModelExample, RefCounted)

public:
    void demonstrate_memory_patterns() {
        // 1. Reference Counted Objects (Automatic cleanup)
        Ref<Resource> resource = memnew(Resource);
        Ref<Texture2D> texture = ResourceLoader::load("res://icon.png");
        // No manual cleanup needed - reference counting handles it

        // 2. Scene Tree Objects (Scene-managed)
        Node* node = memnew(Node);
        add_child(node);  // Scene tree takes ownership
        // node->queue_free() when no longer needed

        // 3. Manually Managed Objects (Rare)
        Object* raw_object = memnew(Object);
        // Must call memdelete(raw_object) manually
        memdelete(raw_object);

        // 4. Stack Objects (Automatic cleanup)
        Vector3 position(1, 2, 3);
        String name = "example";
        // Automatically cleaned up when out of scope
    }
};
```

### Reference Counting Deep Dive

```cpp
class ReferenceCountingGuide : public RefCounted {
    GDCLASS(ReferenceCountingGuide, RefCounted)

private:
    Ref<Resource> owned_resource;
    Vector<Ref<Resource>> resource_list;

public:
    void demonstrate_ref_patterns() {
        // Creating references
        Ref<Resource> res1 = memnew(Resource);
        UtilityFunctions::print("Initial ref count:", res1->get_reference_count());  // 1

        // Copying references (increases count)
        Ref<Resource> res2 = res1;
        UtilityFunctions::print("After copy:", res1->get_reference_count());  // 2

        // Storing in member variables
        owned_resource = res1;
        UtilityFunctions::print("After member assignment:", res1->get_reference_count());  // 3

        // Adding to collections
        resource_list.append(res1);
        UtilityFunctions::print("After array append:", res1->get_reference_count());  // 4

        // Clearing references (decreases count)
        res2 = Ref<Resource>();  // or res2.unref();
        UtilityFunctions::print("After clearing res2:", res1->get_reference_count());  // 3

        // When all refs are cleared, object is automatically deleted
    }

    void demonstrate_weak_references() {
        Ref<Resource> strong_ref = memnew(Resource);

        // Get object ID for weak reference
        ObjectID obj_id = strong_ref->get_instance_id();

        // Clear strong reference
        strong_ref = Ref<Resource>();

        // Try to get object back (will be null if deleted)
        Object* obj = ObjectDB::get_instance(obj_id);
        if (obj) {
            UtilityFunctions::print("Object still exists");
        } else {
            UtilityFunctions::print("Object was deleted");
        }
    }

    Ref<Resource> create_resource_safe() {
        // Good: Return Ref<> for automatic management
        Ref<Resource> resource = memnew(Resource);
        resource->set_name("SafeResource");
        return resource;
    }

    Resource* create_resource_unsafe() {
        // BAD: Returning raw pointer creates memory management issues
        Resource* resource = memnew(Resource);
        resource->set_name("UnsafeResource");
        return resource;  // Caller must handle reference counting manually
    }
};
```

## Custom Resource Classes

**Creating saveable, reference-counted data:** Custom Resource classes are perfect for game data, configuration, and any data that needs to be saved/loaded.

### Basic Custom Resource

```cpp
class PlayerData : public Resource {
    GDCLASS(PlayerData, Resource)

private:
    String player_name = "Player";
    int level = 1;
    int experience = 0;
    Dictionary inventory;
    PackedStringArray completed_quests;

protected:
    static void _bind_methods() {
        // Property bindings for saving/loading
        ClassDB::bind_method(D_METHOD("set_player_name", "name"), &PlayerData::set_player_name);
        ClassDB::bind_method(D_METHOD("get_player_name"), &PlayerData::get_player_name);
        ADD_PROPERTY(PropertyInfo(Variant::STRING, "player_name"),
                     "set_player_name", "get_player_name");

        ClassDB::bind_method(D_METHOD("set_level", "level"), &PlayerData::set_level);
        ClassDB::bind_method(D_METHOD("get_level"), &PlayerData::get_level);
        ADD_PROPERTY(PropertyInfo(Variant::INT, "level"),
                     "set_level", "get_level");

        ClassDB::bind_method(D_METHOD("set_experience", "exp"), &PlayerData::set_experience);
        ClassDB::bind_method(D_METHOD("get_experience"), &PlayerData::get_experience);
        ADD_PROPERTY(PropertyInfo(Variant::INT, "experience"),
                     "set_experience", "get_experience");

        ClassDB::bind_method(D_METHOD("set_inventory", "inv"), &PlayerData::set_inventory);
        ClassDB::bind_method(D_METHOD("get_inventory"), &PlayerData::get_inventory);
        ADD_PROPERTY(PropertyInfo(Variant::DICTIONARY, "inventory"),
                     "set_inventory", "get_inventory");

        ClassDB::bind_method(D_METHOD("set_completed_quests", "quests"), &PlayerData::set_completed_quests);
        ClassDB::bind_method(D_METHOD("get_completed_quests"), &PlayerData::get_completed_quests);
        ADD_PROPERTY(PropertyInfo(Variant::PACKED_STRING_ARRAY, "completed_quests"),
                     "set_completed_quests", "get_completed_quests");

        // Utility methods
        ClassDB::bind_method(D_METHOD("add_experience", "amount"), &PlayerData::add_experience);
        ClassDB::bind_method(D_METHOD("add_item", "item_name", "quantity"), &PlayerData::add_item);
        ClassDB::bind_method(D_METHOD("complete_quest", "quest_name"), &PlayerData::complete_quest);
        ClassDB::bind_method(D_METHOD("is_quest_completed", "quest_name"), &PlayerData::is_quest_completed);
    }

public:
    // Property implementations
    void set_player_name(const String& p_name) { player_name = p_name; }
    String get_player_name() const { return player_name; }

    void set_level(int p_level) { level = MAX(1, p_level); }
    int get_level() const { return level; }

    void set_experience(int p_exp) { experience = MAX(0, p_exp); }
    int get_experience() const { return experience; }

    void set_inventory(const Dictionary& p_inventory) { inventory = p_inventory; }
    Dictionary get_inventory() const { return inventory; }

    void set_completed_quests(const PackedStringArray& p_quests) { completed_quests = p_quests; }
    PackedStringArray get_completed_quests() const { return completed_quests; }

    // Game logic methods
    void add_experience(int amount) {
        experience += amount;

        // Check for level up
        int required_exp = level * 100;  // Simple formula
        if (experience >= required_exp) {
            level++;
            experience -= required_exp;
            UtilityFunctions::print("Level up! Now level:", level);
        }
    }

    void add_item(const String& item_name, int quantity = 1) {
        int current = inventory.get(item_name, 0);
        inventory[item_name] = current + quantity;
    }

    void complete_quest(const String& quest_name) {
        if (!is_quest_completed(quest_name)) {
            completed_quests.append(quest_name);
        }
    }

    bool is_quest_completed(const String& quest_name) const {
        return completed_quests.has(quest_name);
    }

    // Save/load helpers
    void save_to_file(const String& path) {
        Error err = ResourceSaver::save(this, path);
        if (err == OK) {
            UtilityFunctions::print("Player data saved to:", path);
        } else {
            ERR_PRINT("Failed to save player data: " + itos(err));
        }
    }

    static Ref<PlayerData> load_from_file(const String& path) {
        if (!FileAccess::file_exists(path)) {
            // Create default data if file doesn't exist
            return create_default();
        }

        Ref<PlayerData> data = ResourceLoader::load(path);
        if (data.is_null()) {
            ERR_PRINT("Failed to load player data from: " + path);
            return create_default();
        }

        return data;
    }

    static Ref<PlayerData> create_default() {
        Ref<PlayerData> data = memnew(PlayerData);
        data->set_player_name("New Player");
        data->set_level(1);
        data->set_experience(0);

        // Add starting items
        data->add_item("Health Potion", 3);
        data->add_item("Copper Coins", 50);

        return data;
    }
};
```

### Resource with Nested Resources

```cpp
class GameConfiguration : public Resource {
    GDCLASS(GameConfiguration, Resource)

private:
    Ref<PlayerData> default_player_data;
    Dictionary graphics_settings;
    Dictionary audio_settings;
    Dictionary key_bindings;
    Vector<Ref<Resource>> mod_list;

protected:
    static void _bind_methods() {
        // Nested resource
        ClassDB::bind_method(D_METHOD("set_default_player_data", "data"),
                           &GameConfiguration::set_default_player_data);
        ClassDB::bind_method(D_METHOD("get_default_player_data"),
                           &GameConfiguration::get_default_player_data);
        ADD_PROPERTY(PropertyInfo(Variant::OBJECT, "default_player_data",
                                 PROPERTY_HINT_RESOURCE_TYPE, "PlayerData"),
                     "set_default_player_data", "get_default_player_data");

        // Settings dictionaries
        ClassDB::bind_method(D_METHOD("set_graphics_settings", "settings"),
                           &GameConfiguration::set_graphics_settings);
        ClassDB::bind_method(D_METHOD("get_graphics_settings"),
                           &GameConfiguration::get_graphics_settings);
        ADD_PROPERTY(PropertyInfo(Variant::DICTIONARY, "graphics_settings"),
                     "set_graphics_settings", "get_graphics_settings");

        // Resource array
        ClassDB::bind_method(D_METHOD("set_mod_list", "mods"),
                           &GameConfiguration::set_mod_list);
        ClassDB::bind_method(D_METHOD("get_mod_list"),
                           &GameConfiguration::get_mod_list);
        ADD_PROPERTY(PropertyInfo(Variant::ARRAY, "mod_list",
                                 PROPERTY_HINT_ARRAY_TYPE, "Resource"),
                     "set_mod_list", "get_mod_list");

        // Property groups
        ADD_GROUP("Player", "");
        ADD_GROUP("Settings", "");
        ADD_GROUP("Mods", "");
    }

public:
    GameConfiguration() {
        // Initialize with defaults
        default_player_data = PlayerData::create_default();
        setup_default_settings();
    }

    void set_default_player_data(const Ref<PlayerData>& p_data) {
        default_player_data = p_data;
    }

    Ref<PlayerData> get_default_player_data() const {
        return default_player_data;
    }

    void set_graphics_settings(const Dictionary& p_settings) {
        graphics_settings = p_settings;
    }

    Dictionary get_graphics_settings() const {
        return graphics_settings;
    }

    void set_mod_list(const Vector<Ref<Resource>>& p_mods) {
        mod_list = p_mods;
    }

    Vector<Ref<Resource>> get_mod_list() const {
        return mod_list;
    }

    void apply_graphics_settings() {
        // Apply graphics settings to the engine
        if (graphics_settings.has("resolution")) {
            Vector2i resolution = graphics_settings["resolution"];
            // Apply resolution...
        }

        if (graphics_settings.has("fullscreen")) {
            bool fullscreen = graphics_settings["fullscreen"];
            // Apply fullscreen...
        }
    }

private:
    void setup_default_settings() {
        graphics_settings["resolution"] = Vector2i(1920, 1080);
        graphics_settings["fullscreen"] = false;
        graphics_settings["vsync"] = true;
        graphics_settings["shadows"] = true;

        audio_settings["master_volume"] = 1.0;
        audio_settings["music_volume"] = 0.8;
        audio_settings["sfx_volume"] = 1.0;

        key_bindings["move_up"] = "w";
        key_bindings["move_down"] = "s";
        key_bindings["move_left"] = "a";
        key_bindings["move_right"] = "d";
    }
};
```

## Asset Loading and Management

### Efficient Resource Loading

```cpp
class ResourceManager : public RefCounted {
    GDCLASS(ResourceManager, RefCounted)

private:
    HashMap<String, Ref<Resource>> resource_cache;
    HashMap<String, Array> resource_dependencies;
    float cache_cleanup_interval = 30.0f;
    float last_cleanup_time = 0.0f;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("preload_resource", "path"),
                           &ResourceManager::preload_resource);
        ClassDB::bind_method(D_METHOD("get_resource", "path"),
                           &ResourceManager::get_resource);
        ClassDB::bind_method(D_METHOD("unload_resource", "path"),
                           &ResourceManager::unload_resource);
        ClassDB::bind_method(D_METHOD("preload_batch", "paths"),
                           &ResourceManager::preload_batch);
        ClassDB::bind_method(D_METHOD("cleanup_unused"),
                           &ResourceManager::cleanup_unused);
    }

public:
    Ref<Resource> preload_resource(const String& path) {
        // Check cache first
        if (resource_cache.has(path)) {
            return resource_cache[path];
        }

        // Load and cache
        Ref<Resource> resource = ResourceLoader::load(path);
        if (resource.is_valid()) {
            resource_cache[path] = resource;

            // Track dependencies
            PackedStringArray deps = ResourceLoader::get_dependencies(path);
            Array dep_array;
            for (int i = 0; i < deps.size(); i++) {
                dep_array.append(deps[i]);
            }
            resource_dependencies[path] = dep_array;

            UtilityFunctions::print("Loaded and cached:", path);
        } else {
            ERR_PRINT("Failed to load resource: " + path);
        }

        return resource;
    }

    Ref<Resource> get_resource(const String& path) {
        // Return cached resource or load if needed
        if (resource_cache.has(path)) {
            return resource_cache[path];
        }

        return preload_resource(path);
    }

    void unload_resource(const String& path) {
        if (resource_cache.has(path)) {
            resource_cache.erase(path);
            resource_dependencies.erase(path);
            UtilityFunctions::print("Unloaded:", path);
        }
    }

    void preload_batch(const PackedStringArray& paths) {
        for (int i = 0; i < paths.size(); i++) {
            preload_resource(paths[i]);
        }
    }

    void cleanup_unused() {
        Vector<String> to_remove;

        for (const KeyValue<String, Ref<Resource>>& E : resource_cache) {
            // Check if resource is only referenced by our cache
            if (E.value.is_valid() && E.value->get_reference_count() == 1) {
                to_remove.append(E.key);
            }
        }

        for (const String& path : to_remove) {
            unload_resource(path);
        }

        if (to_remove.size() > 0) {
            UtilityFunctions::print("Cleaned up", to_remove.size(), "unused resources");
        }
    }

    void update(float delta) {
        last_cleanup_time += delta;
        if (last_cleanup_time >= cache_cleanup_interval) {
            cleanup_unused();
            last_cleanup_time = 0.0f;
        }
    }

    Dictionary get_cache_stats() {
        Dictionary stats;
        stats["cached_resources"] = resource_cache.size();
        stats["dependencies_tracked"] = resource_dependencies.size();

        int total_references = 0;
        for (const KeyValue<String, Ref<Resource>>& E : resource_cache) {
            if (E.value.is_valid()) {
                total_references += E.value->get_reference_count();
            }
        }
        stats["total_references"] = total_references;

        return stats;
    }
};
```

### Async Resource Loading

```cpp
class AsyncResourceLoader : public RefCounted {
    GDCLASS(AsyncResourceLoader, RefCounted)

private:
    struct LoadRequest {
        String path;
        Callable callback;
        bool completed = false;
        Ref<Resource> result;
    };

    Vector<LoadRequest> pending_requests;
    int max_concurrent_loads = 3;
    int active_loads = 0;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("load_async", "path", "callback"),
                           &AsyncResourceLoader::load_async);
        ClassDB::bind_method(D_METHOD("process_loads"),
                           &AsyncResourceLoader::process_loads);
        ClassDB::bind_method(D_METHOD("get_load_progress", "path"),
                           &AsyncResourceLoader::get_load_progress);
    }

public:
    void load_async(const String& path, const Callable& callback) {
        LoadRequest request;
        request.path = path;
        request.callback = callback;
        pending_requests.append(request);

        // Start loading if we have capacity
        process_loads();
    }

    void process_loads() {
        // Start new loads if capacity available
        while (active_loads < max_concurrent_loads && has_pending_requests()) {
            start_next_load();
        }

        // Check completed loads
        for (int i = pending_requests.size() - 1; i >= 0; i--) {
            LoadRequest& request = pending_requests.write[i];

            if (request.completed) {
                // Call completion callback
                if (request.callback.is_valid()) {
                    request.callback.call(request.result);
                }

                pending_requests.remove_at(i);
                active_loads--;
            } else {
                // Check if load completed
                check_load_status(request);
            }
        }
    }

    float get_load_progress(const String& path) {
        Array progress = ResourceLoader::load_threaded_get_status(path);
        if (progress.size() > 0) {
            return progress[0];  // Progress value 0.0-1.0
        }
        return 0.0f;
    }

private:
    bool has_pending_requests() {
        for (const LoadRequest& request : pending_requests) {
            if (!request.completed && request.result.is_null()) {
                return true;
            }
        }
        return false;
    }

    void start_next_load() {
        for (LoadRequest& request : pending_requests) {
            if (!request.completed && request.result.is_null()) {
                // Start threaded load
                Error err = ResourceLoader::load_threaded_request(request.path);
                if (err == OK) {
                    active_loads++;
                    UtilityFunctions::print("Started async load:", request.path);
                } else {
                    request.completed = true;
                    ERR_PRINT("Failed to start async load: " + request.path);
                }
                break;
            }
        }
    }

    void check_load_status(LoadRequest& request) {
        ResourceLoader::ThreadLoadStatus status = ResourceLoader::load_threaded_get_status(request.path);

        switch (status) {
            case ResourceLoader::THREAD_LOAD_LOADED:
                request.result = ResourceLoader::load_threaded_get(request.path);
                request.completed = true;
                UtilityFunctions::print("Completed async load:", request.path);
                break;

            case ResourceLoader::THREAD_LOAD_FAILED:
                request.completed = true;
                ERR_PRINT("Async load failed: " + request.path);
                break;

            case ResourceLoader::THREAD_LOAD_INVALID_RESOURCE:
                request.completed = true;
                ERR_PRINT("Invalid resource: " + request.path);
                break;

            case ResourceLoader::THREAD_LOAD_IN_PROGRESS:
                // Still loading, continue waiting
                break;
        }
    }
};
```

## Memory Leak Prevention

### Common Memory Leak Patterns

```cpp
class MemoryLeakPrevention : public Node {
    GDCLASS(MemoryLeakPrevention, Node)

private:
    Vector<Node*> child_nodes;
    HashMap<String, Ref<Resource>> resources;

public:
    void demonstrate_leak_patterns() {
        // BAD: Circular references
        create_circular_reference_leak();

        // BAD: Forgotten manual cleanup
        create_manual_cleanup_leak();

        // BAD: Signal connections without cleanup
        create_signal_leak();

        // GOOD: Proper cleanup patterns
        demonstrate_proper_cleanup();
    }

private:
    void create_circular_reference_leak() {
        // BAD: Parent and child both hold strong references to each other
        Ref<RefCounted> parent = memnew(RefCounted);
        Ref<RefCounted> child = memnew(RefCounted);

        // This creates a circular reference that prevents cleanup
        // parent->set_child(child);  // parent holds ref to child
        // child->set_parent(parent);  // child holds ref to parent
        // Neither will be cleaned up!
    }

    void create_manual_cleanup_leak() {
        // BAD: Creating objects that need manual cleanup but forgetting to clean them up
        Node* node = memnew(Node);
        Object* object = memnew(Object);

        // These won't be cleaned up automatically!
        // Should call: memdelete(node); memdelete(object);

        // GOOD: Use RAII or add to scene tree
        Node* managed_node = memnew(Node);
        add_child(managed_node);  // Scene tree will clean up

        // Or use smart pointers for non-Godot objects
    }

    void create_signal_leak() {
        // BAD: Connecting signals without cleanup
        Node* emitter = memnew(Node);

        // This creates a connection that won't be cleaned up
        emitter->connect("tree_entered", Callable(this, "on_node_entered"));

        // If emitter is deleted elsewhere, connection remains dangling

        // GOOD: Always disconnect in cleanup
        child_nodes.append(emitter);  // Track for cleanup
    }

    void demonstrate_proper_cleanup() {
        // GOOD: Use Ref<> for automatic cleanup
        Ref<Resource> resource = memnew(Resource);
        // Automatically cleaned up when ref goes out of scope

        // GOOD: Weak references to break cycles
        WeakRef weak_ref = memnew(WeakRef);
        Node* node = memnew(Node);
        weak_ref->reference = node;
        add_child(node);

        // Check if still valid later
        Node* retrieved = Object::cast_to<Node>(weak_ref->get_ref());
        if (retrieved) {
            // Still valid
        }

        // GOOD: RAII pattern for cleanup
        {
            ResourceCleanupGuard guard(this);
            // Resources are cleaned up when guard goes out of scope
        }
    }

    void _exit_tree() override {
        // Clean up all manually tracked resources
        for (Node* node : child_nodes) {
            if (is_instance_valid(node)) {
                if (node->is_connected("tree_entered", Callable(this, "on_node_entered"))) {
                    node->disconnect("tree_entered", Callable(this, "on_node_entered"));
                }

                if (node->get_parent() != this) {
                    memdelete(node);  // Only if not in scene tree
                }
            }
        }
        child_nodes.clear();

        // Clear resource cache
        resources.clear();
    }

    void on_node_entered() {
        // Signal handler
    }

    // RAII helper class
    class ResourceCleanupGuard {
        MemoryLeakPrevention* owner;

    public:
        ResourceCleanupGuard(MemoryLeakPrevention* p_owner) : owner(p_owner) {}

        ~ResourceCleanupGuard() {
            // Cleanup resources when guard is destroyed
            owner->resources.clear();
        }
    };
};
```

### Memory Profiling and Debugging

```cpp
class MemoryProfiler : public RefCounted {
    GDCLASS(MemoryProfiler, RefCounted)

private:
    HashMap<String, uint64_t> allocation_tracker;
    uint64_t total_allocations = 0;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("track_allocation", "name", "size"),
                           &MemoryProfiler::track_allocation);
        ClassDB::bind_method(D_METHOD("track_deallocation", "name", "size"),
                           &MemoryProfiler::track_deallocation);
        ClassDB::bind_method(D_METHOD("get_memory_report"),
                           &MemoryProfiler::get_memory_report);
        ClassDB::bind_method(D_METHOD("get_system_memory_info"),
                           &MemoryProfiler::get_system_memory_info);
    }

public:
    void track_allocation(const String& name, uint64_t size) {
        allocation_tracker[name] = allocation_tracker.get(name, 0) + size;
        total_allocations += size;
    }

    void track_deallocation(const String& name, uint64_t size) {
        uint64_t current = allocation_tracker.get(name, 0);
        allocation_tracker[name] = (current >= size) ? current - size : 0;
        total_allocations = (total_allocations >= size) ? total_allocations - size : 0;
    }

    Dictionary get_memory_report() {
        Dictionary report;
        report["total_tracked"] = static_cast<int>(total_allocations);

        Dictionary breakdown;
        for (const KeyValue<String, uint64_t>& E : allocation_tracker) {
            breakdown[E.key] = static_cast<int>(E.value);
        }
        report["breakdown"] = breakdown;

        return report;
    }

    Dictionary get_system_memory_info() {
        Dictionary info;

        // Godot memory info
        info["static_memory"] = static_cast<int>(OS::get_singleton()->get_static_memory_usage());
        info["static_peak"] = static_cast<int>(OS::get_singleton()->get_static_memory_peak_usage());

        // Object counts
        info["object_count"] = Engine::get_singleton()->get_process_frames(); // Placeholder

        return info;
    }

    void print_memory_report() {
        Dictionary report = get_memory_report();
        Dictionary system_info = get_system_memory_info();

        UtilityFunctions::print("=== Memory Report ===");
        UtilityFunctions::print("Tracked allocations:", report["total_tracked"]);
        UtilityFunctions::print("System memory:", system_info["static_memory"]);
        UtilityFunctions::print("Peak memory:", system_info["static_peak"]);

        Dictionary breakdown = report["breakdown"];
        UtilityFunctions::print("Breakdown by category:");
        for (const KeyValue<Variant, Variant>& E : breakdown) {
            UtilityFunctions::print("  ", E.key, ":", E.value);
        }
    }
};

// RAII wrapper for automatic memory tracking
template<typename T>
class TrackedAllocation {
private:
    T* ptr;
    String category;
    uint64_t size;
    Ref<MemoryProfiler> profiler;

public:
    TrackedAllocation(const String& p_category, Ref<MemoryProfiler> p_profiler)
        : category(p_category), profiler(p_profiler) {
        ptr = memnew(T);
        size = sizeof(T);
        if (profiler.is_valid()) {
            profiler->track_allocation(category, size);
        }
    }

    ~TrackedAllocation() {
        if (ptr) {
            memdelete(ptr);
            if (profiler.is_valid()) {
                profiler->track_deallocation(category, size);
            }
        }
    }

    T* operator->() { return ptr; }
    T& operator*() { return *ptr; }
    T* get() { return ptr; }

    // Move semantics
    TrackedAllocation(TrackedAllocation&& other) noexcept
        : ptr(other.ptr), category(other.category), size(other.size), profiler(other.profiler) {
        other.ptr = nullptr;
    }

    TrackedAllocation& operator=(TrackedAllocation&& other) noexcept {
        if (this != &other) {
            if (ptr) {
                memdelete(ptr);
                if (profiler.is_valid()) {
                    profiler->track_deallocation(category, size);
                }
            }
            ptr = other.ptr;
            category = other.category;
            size = other.size;
            profiler = other.profiler;
            other.ptr = nullptr;
        }
        return *this;
    }

    // Prevent copying
    TrackedAllocation(const TrackedAllocation&) = delete;
    TrackedAllocation& operator=(const TrackedAllocation&) = delete;
};
```

## Best Practices Summary

### 1. Choose Appropriate Memory Management
- Use `Ref<>` for reference-counted objects
- Use scene tree ownership for nodes
- Avoid manual memory management when possible

### 2. Resource Loading Strategies
- Cache frequently used resources
- Use async loading for large assets
- Implement cleanup for unused resources

### 3. Prevent Memory Leaks
- Break circular references
- Clean up signal connections
- Use RAII patterns where possible

### 4. Monitor Memory Usage
- Track allocations in debug builds
- Profile memory usage patterns
- Set up automated leak detection

## Conclusion

Effective resource management in GDExtension requires understanding Godot's memory model, using appropriate reference patterns, and implementing proper cleanup strategies. By following these patterns, you can create extensions that are both performant and stable, avoiding common pitfalls like memory leaks and dangling references.
