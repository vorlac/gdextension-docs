# Signal Patterns - Event-Driven Communication

## Introduction

**Mastering signals for clean, decoupled architecture:** Signals are Godot's event system, enabling loose coupling between components. This guide covers signal patterns, best practices, and advanced techniques for building robust event-driven systems in GDExtension.

## Understanding Signals

**Event-driven communication without tight coupling:** Signals allow objects to notify others about events without knowing who is listening. This creates flexible, maintainable architectures where components can interact without direct dependencies.

### Basic Signal Concepts

```cpp
class SignalBasicsExample : public Node {
    GDCLASS(SignalBasicsExample, Node)

private:
    int health = 100;
    bool is_alive = true;

protected:
    static void _bind_methods() {
        // Define signals with clear names and parameters
        ADD_SIGNAL(MethodInfo("health_changed", 
                             PropertyInfo(Variant::INT, "new_health"),
                             PropertyInfo(Variant::INT, "old_health")));
        
        ADD_SIGNAL(MethodInfo("died"));
        
        ADD_SIGNAL(MethodInfo("status_updated",
                             PropertyInfo(Variant::STRING, "status"),
                             PropertyInfo(Variant::DICTIONARY, "details")));

        // Methods for signal interaction
        ClassDB::bind_method(D_METHOD("take_damage", "amount"), &SignalBasicsExample::take_damage);
        ClassDB::bind_method(D_METHOD("heal", "amount"), &SignalBasicsExample::heal);
    }

public:
    void _ready() override {
        // Connect to own signals for internal logic
        connect("health_changed", Callable(this, "_on_health_changed"));
        connect("died", Callable(this, "_on_died"));
        
        UtilityFunctions::print("SignalBasicsExample ready with health:", health);
    }

    void take_damage(int amount) {
        if (!is_alive) return;
        
        int old_health = health;
        health = MAX(0, health - amount);
        
        // Always emit health_changed signal
        emit_signal("health_changed", health, old_health);
        
        // Emit died signal if health reaches zero
        if (health == 0 && is_alive) {
            is_alive = false;
            emit_signal("died");
        }
        
        // Emit status update with details
        Dictionary details;
        details["damage_taken"] = amount;
        details["remaining_health"] = health;
        emit_signal("status_updated", "damaged", details);
    }

    void heal(int amount) {
        if (!is_alive) return;
        
        int old_health = health;
        health = MIN(100, health + amount);
        
        if (health != old_health) {
            emit_signal("health_changed", health, old_health);
            
            Dictionary details;
            details["healing_received"] = amount;
            details["current_health"] = health;
            emit_signal("status_updated", "healed", details);
        }
    }

private:
    void _on_health_changed(int new_health, int old_health) {
        UtilityFunctions::print("Health changed from", old_health, "to", new_health);
    }

    void _on_died() {
        UtilityFunctions::print("Entity has died!");
        set_process(false);  // Stop processing when dead
    }
};
```

### Signal Connection Patterns

```cpp
class SignalConnectionPatterns : public Node {
    GDCLASS(SignalConnectionPatterns, Node)

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("demonstrate_connection_patterns"), 
                           &SignalConnectionPatterns::demonstrate_connection_patterns);
        ClassDB::bind_method(D_METHOD("on_signal_received", "data"), 
                           &SignalConnectionPatterns::on_signal_received);
    }

public:
    void demonstrate_connection_patterns() {
        Node* emitter = get_node_or_null(NodePath("SignalEmitter"));
        if (!emitter) return;

        // 1. Basic connection
        emitter->connect("basic_signal", Callable(this, "on_signal_received"));

        // 2. One-shot connection (auto-disconnects after first call)
        emitter->connect("one_shot_signal", Callable(this, "on_signal_received"), CONNECT_ONE_SHOT);

        // 3. Deferred connection (calls on idle frame)
        emitter->connect("deferred_signal", Callable(this, "on_signal_received"), CONNECT_DEFERRED);

        // 4. Connection with binding (pass additional data)
        Callable bound_callable = Callable(this, "on_signal_received").bind("extra_data");
        emitter->connect("bound_signal", bound_callable);

        // 5. Lambda connection (if using CustomCallable)
        auto lambda_callback = [this](const Variant &data) {
            UtilityFunctions::print("Lambda received:", data);
        };
        emitter->connect("lambda_signal", Callable::create(lambda_callback));

        // 6. Conditional connection check
        if (!emitter->is_connected("checked_signal", Callable(this, "on_signal_received"))) {
            emitter->connect("checked_signal", Callable(this, "on_signal_received"));
        }
    }

    void on_signal_received(const Variant &data) {
        UtilityFunctions::print("Signal received with data:", data);
    }

    void _exit_tree() override {
        // Clean up connections when exiting
        Node* emitter = get_node_or_null(NodePath("SignalEmitter"));
        if (emitter && is_instance_valid(emitter)) {
            // Disconnect all our connections
            if (emitter->is_connected("basic_signal", Callable(this, "on_signal_received"))) {
                emitter->disconnect("basic_signal", Callable(this, "on_signal_received"));
            }
            // ... disconnect other signals
        }
    }
};
```

## Advanced Signal Patterns

### Event Bus Pattern

```cpp
class EventBus : public Node {
    GDCLASS(EventBus, Node)

private:
    static EventBus* instance;
    HashMap<String, Vector<Callable>> event_listeners;
    Dictionary event_data_cache;

protected:
    static void _bind_methods() {
        ClassDB::bind_static_method("EventBus", D_METHOD("get_instance"), &EventBus::get_instance);
        ClassDB::bind_method(D_METHOD("subscribe", "event_name", "callback"), &EventBus::subscribe);
        ClassDB::bind_method(D_METHOD("unsubscribe", "event_name", "callback"), &EventBus::unsubscribe);
        ClassDB::bind_method(D_METHOD("emit_event", "event_name", "data"), &EventBus::emit_event);
        ClassDB::bind_method(D_METHOD("has_listeners", "event_name"), &EventBus::has_listeners);
        
        // Global events
        ADD_SIGNAL(MethodInfo("global_event", 
                             PropertyInfo(Variant::STRING, "event_name"),
                             PropertyInfo(Variant::DICTIONARY, "event_data")));
    }

public:
    static EventBus* get_instance() {
        if (!instance) {
            instance = memnew(EventBus);
            // Add to tree so it persists
            Engine::get_singleton()->get_main_loop()->get_current_scene()->add_child(instance);
        }
        return instance;
    }

    void subscribe(const String& event_name, const Callable& callback) {
        if (!event_listeners.has(event_name)) {
            event_listeners[event_name] = Vector<Callable>();
        }
        
        Vector<Callable>& listeners = event_listeners[event_name];
        
        // Avoid duplicate subscriptions
        for (const Callable& existing : listeners) {
            if (existing.get_object() == callback.get_object() && 
                existing.get_method() == callback.get_method()) {
                return;  // Already subscribed
            }
        }
        
        listeners.append(callback);
        UtilityFunctions::print("Subscribed to event:", event_name);
    }

    void unsubscribe(const String& event_name, const Callable& callback) {
        if (!event_listeners.has(event_name)) {
            return;
        }
        
        Vector<Callable>& listeners = event_listeners[event_name];
        for (int i = listeners.size() - 1; i >= 0; i--) {
            if (listeners[i].get_object() == callback.get_object() && 
                listeners[i].get_method() == callback.get_method()) {
                listeners.remove_at(i);
                UtilityFunctions::print("Unsubscribed from event:", event_name);
                break;
            }
        }
        
        // Clean up empty listener lists
        if (listeners.is_empty()) {
            event_listeners.erase(event_name);
        }
    }

    void emit_event(const String& event_name, const Dictionary& data = Dictionary()) {
        // Cache the event data
        event_data_cache[event_name] = data;
        
        // Emit to direct subscribers
        if (event_listeners.has(event_name)) {
            const Vector<Callable>& listeners = event_listeners[event_name];
            
            for (int i = 0; i < listeners.size(); i++) {
                const Callable& callback = listeners[i];
                
                // Check if the callback is still valid
                if (callback.is_valid()) {
                    callback.call(data);
                } else {
                    // Remove invalid callbacks
                    Vector<Callable>& mutable_listeners = event_listeners[event_name];
                    mutable_listeners.remove_at(i);
                    i--; // Adjust index after removal
                }
            }
        }
        
        // Also emit as global signal for broad listeners
        emit_signal("global_event", event_name, data);
        
        UtilityFunctions::print("Emitted event:", event_name, "to", get_listener_count(event_name), "listeners");
    }

    bool has_listeners(const String& event_name) const {
        return event_listeners.has(event_name) && !event_listeners[event_name].is_empty();
    }

    int get_listener_count(const String& event_name) const {
        if (event_listeners.has(event_name)) {
            return event_listeners[event_name].size();
        }
        return 0;
    }

    Dictionary get_cached_event_data(const String& event_name) const {
        if (event_data_cache.has(event_name)) {
            return event_data_cache[event_name];
        }
        return Dictionary();
    }

    // Utility methods for common game events
    void emit_player_event(const String& action, const Dictionary& details = Dictionary()) {
        Dictionary data = details;
        data["category"] = "player";
        data["action"] = action;
        data["timestamp"] = OS::get_singleton()->get_unix_time();
        emit_event("player." + action, data);
    }

    void emit_game_state_event(const String& state, const Dictionary& details = Dictionary()) {
        Dictionary data = details;
        data["category"] = "game_state";
        data["new_state"] = state;
        data["timestamp"] = OS::get_singleton()->get_unix_time();
        emit_event("game_state." + state, data);
    }

    void emit_ui_event(const String& action, const Dictionary& details = Dictionary()) {
        Dictionary data = details;
        data["category"] = "ui";
        data["action"] = action;
        emit_event("ui." + action, data);
    }

    void _exit_tree() override {
        // Clean up when exiting
        event_listeners.clear();
        event_data_cache.clear();
        instance = nullptr;
    }
};

EventBus* EventBus::instance = nullptr;
```

### Signal Chain Pattern

```cpp
class SignalChain : public RefCounted {
    GDCLASS(SignalChain, RefCounted)

private:
    struct ChainLink {
        Callable processor;
        bool async = false;
        float delay = 0.0f;
    };
    
    Vector<ChainLink> chain;
    int current_index = 0;
    bool processing = false;
    Variant current_data;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("add_processor", "processor", "async", "delay"), 
                           &SignalChain::add_processor, DEFVAL(false), DEFVAL(0.0));
        ClassDB::bind_method(D_METHOD("execute", "initial_data"), &SignalChain::execute);
        ClassDB::bind_method(D_METHOD("clear_chain"), &SignalChain::clear_chain);
        
        ADD_SIGNAL(MethodInfo("chain_completed", PropertyInfo(Variant::NIL, "final_data")));
        ADD_SIGNAL(MethodInfo("chain_failed", PropertyInfo(Variant::STRING, "error")));
        ADD_SIGNAL(MethodInfo("step_completed", PropertyInfo(Variant::INT, "step"), PropertyInfo(Variant::NIL, "data")));
    }

public:
    void add_processor(const Callable& processor, bool async = false, float delay = 0.0f) {
        ChainLink link;
        link.processor = processor;
        link.async = async;
        link.delay = delay;
        chain.append(link);
    }

    void execute(const Variant& initial_data) {
        if (processing) {
            WARN_PRINT("SignalChain already processing, ignoring new execution");
            return;
        }
        
        current_data = initial_data;
        current_index = 0;
        processing = true;
        
        process_next_step();
    }

    void clear_chain() {
        chain.clear();
        current_index = 0;
        processing = false;
    }

private:
    void process_next_step() {
        if (current_index >= chain.size()) {
            // Chain completed
            processing = false;
            emit_signal("chain_completed", current_data);
            return;
        }
        
        const ChainLink& link = chain[current_index];
        
        if (link.delay > 0.0f) {
            // Schedule delayed execution
            SceneTree* tree = Engine::get_singleton()->get_main_loop();
            if (tree) {
                tree->create_timer(link.delay)->timeout.connect(Callable(this, "execute_current_step"));
                return;
            }
        }
        
        execute_current_step();
    }

    void execute_current_step() {
        if (current_index >= chain.size()) return;
        
        const ChainLink& link = chain[current_index];
        
        try {
            if (link.async) {
                // Async processing - assume processor will call step_completed
                link.processor.call(current_data, Callable(this, "on_async_step_completed"));
            } else {
                // Sync processing
                Variant result = link.processor.call(current_data);
                current_data = result;
                
                emit_signal("step_completed", current_index, current_data);
                current_index++;
                
                // Continue to next step
                call_deferred("process_next_step");
            }
        } catch (...) {
            processing = false;
            emit_signal("chain_failed", "Error executing step " + itos(current_index));
        }
    }

    void on_async_step_completed(const Variant& result) {
        current_data = result;
        emit_signal("step_completed", current_index, current_data);
        current_index++;
        
        // Continue to next step
        process_next_step();
    }
};

// Example usage of SignalChain
class ChainExample : public Node {
    GDCLASS(ChainExample, Node)

private:
    Ref<SignalChain> data_processing_chain;

public:
    void _ready() override {
        data_processing_chain = memnew(SignalChain);
        
        // Build processing chain
        data_processing_chain->add_processor(Callable(this, "validate_data"));
        data_processing_chain->add_processor(Callable(this, "transform_data"));
        data_processing_chain->add_processor(Callable(this, "save_data"), true);  // Async step
        
        // Connect to completion
        data_processing_chain->connect("chain_completed", Callable(this, "on_processing_complete"));
        data_processing_chain->connect("chain_failed", Callable(this, "on_processing_failed"));
        
        // Start processing
        Dictionary initial_data;
        initial_data["raw_input"] = "example data";
        data_processing_chain->execute(initial_data);
    }

    Variant validate_data(const Variant& input) {
        UtilityFunctions::print("Validating data:", input);
        // Validation logic...
        return input;  // Return validated data
    }

    Variant transform_data(const Variant& input) {
        UtilityFunctions::print("Transforming data:", input);
        // Transformation logic...
        Dictionary transformed = input;
        transformed["processed"] = true;
        return transformed;
    }

    void save_data(const Variant& input, const Callable& completion_callback) {
        UtilityFunctions::print("Saving data asynchronously:", input);
        
        // Simulate async save operation
        SceneTree* tree = Engine::get_singleton()->get_main_loop();
        if (tree) {
            tree->create_timer(1.0)->timeout.connect([completion_callback, input]() {
                Dictionary result = input;
                result["saved"] = true;
                completion_callback.call(result);
            });
        }
    }

    void on_processing_complete(const Variant& final_data) {
        UtilityFunctions::print("Processing chain completed:", final_data);
    }

    void on_processing_failed(const String& error) {
        ERR_PRINT("Processing chain failed: " + error);
    }
};
```

## Signal Safety and Error Handling

### Safe Signal Connections

```cpp
class SafeSignalManager : public Node {
    GDCLASS(SafeSignalManager, Node)

private:
    struct ConnectionInfo {
        Object* emitter;
        String signal_name;
        Callable callback;
        ObjectID emitter_id;
    };
    
    Vector<ConnectionInfo> managed_connections;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("connect_safe", "emitter", "signal_name", "callback"), 
                           &SafeSignalManager::connect_safe);
        ClassDB::bind_method(D_METHOD("disconnect_all"), &SafeSignalManager::disconnect_all);
        ClassDB::bind_method(D_METHOD("cleanup_invalid_connections"), 
                           &SafeSignalManager::cleanup_invalid_connections);
    }

public:
    void connect_safe(Object* emitter, const String& signal_name, const Callable& callback) {
        if (!emitter || !callback.is_valid()) {
            ERR_PRINT("Invalid emitter or callback for signal connection");
            return;
        }
        
        // Check if signal exists
        if (!emitter->has_signal(signal_name)) {
            WARN_PRINT("Signal '" + signal_name + "' does not exist on object");
            return;
        }
        
        // Check if already connected to avoid duplicate connections
        if (emitter->is_connected(signal_name, callback)) {
            WARN_PRINT("Signal already connected: " + signal_name);
            return;
        }
        
        // Make the connection
        Error err = emitter->connect(signal_name, callback);
        if (err != OK) {
            ERR_PRINT("Failed to connect signal: " + signal_name + " (Error: " + itos(err) + ")");
            return;
        }
        
        // Track the connection
        ConnectionInfo info;
        info.emitter = emitter;
        info.signal_name = signal_name;
        info.callback = callback;
        info.emitter_id = emitter->get_instance_id();
        managed_connections.append(info);
        
        UtilityFunctions::print("Safely connected signal:", signal_name);
    }

    void disconnect_all() {
        for (int i = managed_connections.size() - 1; i >= 0; i--) {
            const ConnectionInfo& info = managed_connections[i];
            
            // Check if emitter still exists
            Object* emitter = ObjectDB::get_instance(info.emitter_id);
            if (emitter && emitter->is_connected(info.signal_name, info.callback)) {
                emitter->disconnect(info.signal_name, info.callback);
            }
            
            managed_connections.remove_at(i);
        }
        
        UtilityFunctions::print("Disconnected all managed signals");
    }

    void cleanup_invalid_connections() {
        for (int i = managed_connections.size() - 1; i >= 0; i--) {
            const ConnectionInfo& info = managed_connections[i];
            
            Object* emitter = ObjectDB::get_instance(info.emitter_id);
            if (!emitter || !emitter->is_connected(info.signal_name, info.callback)) {
                managed_connections.remove_at(i);
            }
        }
    }

    void _exit_tree() override {
        disconnect_all();
    }

    int get_connection_count() const {
        return managed_connections.size();
    }

    void print_connection_status() {
        UtilityFunctions::print("=== Connection Status ===");
        for (const ConnectionInfo& info : managed_connections) {
            Object* emitter = ObjectDB::get_instance(info.emitter_id);
            String status = emitter ? "VALID" : "INVALID";
            UtilityFunctions::print("Signal:", info.signal_name, "Status:", status);
        }
    }
};
```

### Error-Resilient Signal Handlers

```cpp
class ResilientSignalHandler : public Node {
    GDCLASS(ResilientSignalHandler, Node)

private:
    int error_count = 0;
    int max_errors = 10;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("handle_risky_signal", "data"), 
                           &ResilientSignalHandler::handle_risky_signal);
        ClassDB::bind_method(D_METHOD("handle_with_fallback", "data"), 
                           &ResilientSignalHandler::handle_with_fallback);
        ClassDB::bind_method(D_METHOD("handle_with_retry", "data", "retry_count"), 
                           &ResilientSignalHandler::handle_with_retry);
    }

public:
    void handle_risky_signal(const Variant& data) {
        try {
            // Risky operation that might fail
            process_unreliable_data(data);
            
        } catch (const std::exception& e) {
            error_count++;
            ERR_PRINT("Signal handler error: " + String(e.what()));
            
            // Disable handler if too many errors
            if (error_count >= max_errors) {
                WARN_PRINT("Too many errors, disabling risky signal handler");
                // Disconnect or disable handler
            }
            
        } catch (...) {
            error_count++;
            ERR_PRINT("Unknown error in signal handler");
        }
    }

    void handle_with_fallback(const Variant& data) {
        bool success = false;
        
        // Try primary handler
        try {
            process_primary_method(data);
            success = true;
        } catch (...) {
            WARN_PRINT("Primary handler failed, trying fallback");
        }
        
        // Try fallback if primary failed
        if (!success) {
            try {
                process_fallback_method(data);
                success = true;
            } catch (...) {
                ERR_PRINT("Both primary and fallback handlers failed");
            }
        }
        
        if (!success) {
            // Last resort - safe minimal handling
            process_safe_minimal(data);
        }
    }

    void handle_with_retry(const Variant& data, int retry_count = 3) {
        for (int attempt = 0; attempt < retry_count; attempt++) {
            try {
                process_unreliable_operation(data);
                return;  // Success, exit retry loop
                
            } catch (...) {
                if (attempt == retry_count - 1) {
                    ERR_PRINT("Signal handler failed after " + itos(retry_count) + " attempts");
                    // Could emit a failure signal here
                } else {
                    WARN_PRINT("Retrying signal handler, attempt " + itos(attempt + 1));
                    // Optional: small delay between retries
                }
            }
        }
    }

private:
    void process_unreliable_data(const Variant& data) {
        // Simulate unreliable processing
        if (Math::randf() < 0.3) {  // 30% failure rate
            throw std::runtime_error("Random processing failure");
        }
        UtilityFunctions::print("Processed data successfully:", data);
    }

    void process_primary_method(const Variant& data) {
        // Primary processing method
        UtilityFunctions::print("Primary processing:", data);
    }

    void process_fallback_method(const Variant& data) {
        // Safer fallback method
        UtilityFunctions::print("Fallback processing:", data);
    }

    void process_safe_minimal(const Variant& data) {
        // Minimal safe processing
        UtilityFunctions::print("Safe minimal processing:", data);
    }

    void process_unreliable_operation(const Variant& data) {
        // Operation that might need retrying
        UtilityFunctions::print("Unreliable operation:", data);
    }
};
```

## Performance Optimization

### Signal Performance Patterns

```cpp
class SignalPerformanceOptimizer : public Node {
    GDCLASS(SignalPerformanceOptimizer, Node)

private:
    // Batching signals
    Vector<Dictionary> pending_events;
    float batch_interval = 0.1f;
    float batch_timer = 0.0f;
    
    // Throttling signals
    HashMap<String, float> last_emission_time;
    float throttle_interval = 0.016f;  // ~60 FPS

protected:
    static void _bind_methods() {
        ADD_SIGNAL(MethodInfo("batch_processed", PropertyInfo(Variant::ARRAY, "events")));
        ADD_SIGNAL(MethodInfo("throttled_update", PropertyInfo(Variant::DICTIONARY, "data")));
        
        ClassDB::bind_method(D_METHOD("queue_event", "event_data"), 
                           &SignalPerformanceOptimizer::queue_event);
        ClassDB::bind_method(D_METHOD("emit_throttled", "signal_name", "data"), 
                           &SignalPerformanceOptimizer::emit_throttled);
    }

public:
    void _process(double delta) override {
        batch_timer += delta;
        
        // Process batched events
        if (batch_timer >= batch_interval && !pending_events.is_empty()) {
            Array events;
            for (const Dictionary& event : pending_events) {
                events.append(event);
            }
            
            emit_signal("batch_processed", events);
            pending_events.clear();
            batch_timer = 0.0f;
        }
    }

    void queue_event(const Dictionary& event_data) {
        // Batch events to reduce signal spam
        pending_events.append(event_data);
        
        // Force flush if batch is getting large
        if (pending_events.size() >= 100) {
            Array events;
            for (const Dictionary& event : pending_events) {
                events.append(event);
            }
            
            emit_signal("batch_processed", events);
            pending_events.clear();
            batch_timer = 0.0f;
        }
    }

    void emit_throttled(const String& signal_name, const Dictionary& data) {
        float current_time = Engine::get_singleton()->get_process_frames() * 0.016f;  // Approximate time
        
        // Check if enough time has passed since last emission
        if (last_emission_time.has(signal_name)) {
            float time_since_last = current_time - last_emission_time[signal_name];
            if (time_since_last < throttle_interval) {
                return;  // Skip emission, too frequent
            }
        }
        
        // Emit the signal
        emit_signal("throttled_update", data);
        last_emission_time[signal_name] = current_time;
    }

    // Signal connection pooling
    void setup_optimized_connections() {
        // Use deferred connections for non-critical signals
        connect("batch_processed", Callable(this, "handle_batch_deferred"), CONNECT_DEFERRED);
        
        // Use direct connections for critical signals
        connect("throttled_update", Callable(this, "handle_critical_immediate"));
    }

private:
    void handle_batch_deferred(const Array& events) {
        // Process batched events on next frame
        for (int i = 0; i < events.size(); i++) {
            Dictionary event = events[i];
            // Process each event...
        }
    }

    void handle_critical_immediate(const Dictionary& data) {
        // Handle immediately for critical updates
        UtilityFunctions::print("Critical update:", data);
    }
};
```

## Testing Signal Systems

### Signal Testing Framework

```cpp
class SignalTester : public RefCounted {
    GDCLASS(SignalTester, RefCounted)

private:
    struct SignalExpectation {
        String signal_name;
        Array expected_args;
        bool received = false;
        Array actual_args;
        int call_count = 0;
    };
    
    Vector<SignalExpectation> expectations;
    Object* test_object = nullptr;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("expect_signal", "object", "signal_name", "args"), 
                           &SignalTester::expect_signal);
        ClassDB::bind_method(D_METHOD("verify_expectations"), 
                           &SignalTester::verify_expectations);
        ClassDB::bind_method(D_METHOD("reset"), &SignalTester::reset);
    }

public:
    void expect_signal(Object* object, const String& signal_name, const Array& expected_args = Array()) {
        if (!object) return;
        
        SignalExpectation expectation;
        expectation.signal_name = signal_name;
        expectation.expected_args = expected_args;
        expectations.append(expectation);
        
        // Connect to the signal to capture it
        Callable capture_callback = Callable(this, "capture_signal").bind(expectations.size() - 1);
        object->connect(signal_name, capture_callback);
        
        test_object = object;
    }

    bool verify_expectations() {
        bool all_passed = true;
        
        for (const SignalExpectation& expectation : expectations) {
            if (!expectation.received) {
                UtilityFunctions::print("FAIL: Signal not received:", expectation.signal_name);
                all_passed = false;
                continue;
            }
            
            if (expectation.expected_args.size() > 0) {
                if (expectation.actual_args != expectation.expected_args) {
                    UtilityFunctions::print("FAIL: Signal args mismatch for:", expectation.signal_name);
                    UtilityFunctions::print("Expected:", expectation.expected_args);
                    UtilityFunctions::print("Actual:", expectation.actual_args);
                    all_passed = false;
                }
            }
            
            UtilityFunctions::print("PASS: Signal received correctly:", expectation.signal_name);
        }
        
        return all_passed;
    }

    void reset() {
        // Disconnect all signal connections
        if (test_object && is_instance_valid(test_object)) {
            for (int i = 0; i < expectations.size(); i++) {
                const SignalExpectation& exp = expectations[i];
                Callable capture_callback = Callable(this, "capture_signal").bind(i);
                if (test_object->is_connected(exp.signal_name, capture_callback)) {
                    test_object->disconnect(exp.signal_name, capture_callback);
                }
            }
        }
        
        expectations.clear();
        test_object = nullptr;
    }

private:
    void capture_signal(int expectation_index, const Variant& arg1 = Variant(), 
                       const Variant& arg2 = Variant(), const Variant& arg3 = Variant()) {
        if (expectation_index < 0 || expectation_index >= expectations.size()) {
            return;
        }
        
        SignalExpectation& expectation = expectations.write[expectation_index];
        expectation.received = true;
        expectation.call_count++;
        
        // Capture arguments
        Array args;
        if (arg1.get_type() != Variant::NIL) args.append(arg1);
        if (arg2.get_type() != Variant::NIL) args.append(arg2);
        if (arg3.get_type() != Variant::NIL) args.append(arg3);
        expectation.actual_args = args;
    }
};

// Example test usage
class SignalTestExample : public Node {
    GDCLASS(SignalTestExample, Node)

public:
    void run_signal_tests() {
        Ref<SignalTester> tester = memnew(SignalTester);
        
        // Create test object
        SignalBasicsExample* test_obj = memnew(SignalBasicsExample);
        add_child(test_obj);
        
        // Set up expectations
        Array health_args = Array::make(90, 100);  // new_health, old_health
        tester->expect_signal(test_obj, "health_changed", health_args);
        
        // Trigger the signals
        test_obj->take_damage(10);
        
        // Verify results
        bool passed = tester->verify_expectations();
        UtilityFunctions::print("Signal tests passed:", passed);
        
        tester->reset();
    }
};
```

## Best Practices Summary

### 1. Signal Design
- Use clear, descriptive signal names
- Include relevant parameters
- Design for loose coupling

### 2. Connection Management
- Always clean up connections
- Use safe connection patterns
- Handle object lifecycle properly

### 3. Performance Optimization
- Batch frequent signals
- Throttle high-frequency emissions
- Use appropriate connection flags

### 4. Error Handling
- Implement resilient handlers
- Provide fallback mechanisms
- Log and handle failures gracefully

## Conclusion

Effective signal usage in GDExtension enables:
- **Loose coupling** between components
- **Event-driven architectures** that are maintainable
- **Performance optimization** through batching and throttling
- **Robust error handling** with fallbacks and retries
- **Testable systems** with signal verification

Signals are a powerful tool for creating flexible, maintainable GDExtension code that integrates seamlessly with Godot's event-driven architecture.