# Thread Safety - Concurrency in GDExtension

## Introduction

**Building thread-safe GDExtension code:** GDExtension runs in a multi-threaded environment where the main thread handles rendering and UI while background threads can process physics, audio, and custom logic. Understanding thread safety is crucial for building robust extensions that don't cause crashes or data corruption.

## Understanding Godot's Threading Model

**Godot's thread architecture:** Godot uses multiple threads for different purposes, and GDExtension code can be called from any of these threads. Understanding which thread your code runs on is essential for thread safety.

### Thread Types in Godot

```cpp
class ThreadAwareExtension : public Node {
    GDCLASS(ThreadAwareExtension, Node)

private:
    std::thread::id main_thread_id;
    std::thread::id render_thread_id;
    std::mutex debug_mutex;

public:
    ThreadAwareExtension() {
        main_thread_id = std::this_thread::get_id();
    }

    void _ready() override {
        print_thread_info("_ready called");
    }

    void _process(double delta) override {
        print_thread_info("_process called");
    }

    void _physics_process(double delta) override {
        print_thread_info("_physics_process called");
    }

private:
    void print_thread_info(const String &context) {
        std::lock_guard<std::mutex> lock(debug_mutex);

        std::thread::id current_id = std::this_thread::get_id();
        String thread_type = "Unknown";

        if (current_id == main_thread_id) {
            thread_type = "Main";
        } else if (OS::get_singleton()->is_in_low_processor_usage_mode()) {
            thread_type = "Background";
        }

        UtilityFunctions::print(vformat("%s on thread: %s (ID: %d)",
                                       context, thread_type,
                                       std::hash<std::thread::id>{}(current_id)));
    }
};
```

### Thread Context Detection

```cpp
class ThreadContextManager : public RefCounted {
    GDCLASS(ThreadContextManager, RefCounted)

private:
    static thread_local ThreadContext current_context;

public:
    enum ThreadContext {
        MAIN_THREAD,
        RENDER_THREAD,
        PHYSICS_THREAD,
        AUDIO_THREAD,
        WORKER_THREAD,
        UNKNOWN_THREAD
    };

    static ThreadContext get_current_context() {
        // First time initialization for this thread
        if (current_context == UNKNOWN_THREAD) {
            detect_thread_context();
        }
        return current_context;
    }

    static bool is_main_thread() {
        return get_current_context() == MAIN_THREAD;
    }

    static bool is_render_thread() {
        return get_current_context() == RENDER_THREAD;
    }

    static void assert_main_thread(const String &operation) {
        if (!is_main_thread()) {
            ERR_PRINT(vformat("Operation '%s' must be called from main thread", operation));
        }
    }

private:
    static void detect_thread_context() {
        // Use Godot APIs that are thread-specific to detect context
        if (Engine::get_singleton() && !OS::get_singleton()->is_stdout_verbose()) {
            current_context = MAIN_THREAD;
        } else {
            // More sophisticated detection based on call stack or thread name
            current_context = WORKER_THREAD;
        }
    }
};

thread_local ThreadContextManager::ThreadContext
    ThreadContextManager::current_context = ThreadContextManager::UNKNOWN_THREAD;
```

## Thread-Safe Data Structures

### Atomic Operations

```cpp
class AtomicCounter : public RefCounted {
    GDCLASS(AtomicCounter, RefCounted)

private:
    std::atomic<int> counter{0};
    std::atomic<double> sum{0.0};
    std::atomic<bool> enabled{true};

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("increment"), &AtomicCounter::increment);
        ClassDB::bind_method(D_METHOD("get_count"), &AtomicCounter::get_count);
        ClassDB::bind_method(D_METHOD("add_value", "value"), &AtomicCounter::add_value);
    }

public:
    void increment() {
        // Thread-safe increment
        counter.fetch_add(1, std::memory_order_relaxed);
    }

    int get_count() const {
        return counter.load(std::memory_order_relaxed);
    }

    void add_value(double value) {
        // Thread-safe addition for floating point
        double expected = sum.load();
        while (!sum.compare_exchange_weak(expected, expected + value,
                                         std::memory_order_relaxed)) {
            // Retry if another thread modified the value
        }
    }

    void set_enabled(bool p_enabled) {
        enabled.store(p_enabled, std::memory_order_release);
    }

    bool is_enabled() const {
        return enabled.load(std::memory_order_acquire);
    }

    // Advanced: Compare and swap operations
    bool try_increment_if_less_than(int threshold) {
        int current = counter.load();
        while (current < threshold) {
            if (counter.compare_exchange_weak(current, current + 1)) {
                return true;  // Successfully incremented
            }
            // current was updated by compare_exchange_weak, retry
        }
        return false;  // Counter was >= threshold
    }
};
```

### Thread-Safe Collections

```cpp
template<typename T>
class ThreadSafeQueue {
private:
    std::queue<T> queue;
    mutable std::mutex mutex;
    std::condition_variable condition;

public:
    void push(const T& item) {
        std::lock_guard<std::mutex> lock(mutex);
        queue.push(item);
        condition.notify_one();
    }

    void push(T&& item) {
        std::lock_guard<std::mutex> lock(mutex);
        queue.push(std::move(item));
        condition.notify_one();
    }

    bool try_pop(T& item) {
        std::lock_guard<std::mutex> lock(mutex);
        if (queue.empty()) {
            return false;
        }
        item = queue.front();
        queue.pop();
        return true;
    }

    T wait_and_pop() {
        std::unique_lock<std::mutex> lock(mutex);
        condition.wait(lock, [this] { return !queue.empty(); });
        T item = queue.front();
        queue.pop();
        return item;
    }

    bool wait_for_pop(T& item, std::chrono::milliseconds timeout) {
        std::unique_lock<std::mutex> lock(mutex);
        if (condition.wait_for(lock, timeout, [this] { return !queue.empty(); })) {
            item = queue.front();
            queue.pop();
            return true;
        }
        return false;
    }

    size_t size() const {
        std::lock_guard<std::mutex> lock(mutex);
        return queue.size();
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mutex);
        return queue.empty();
    }
};

class AsyncTaskProcessor : public Node {
    GDCLASS(AsyncTaskProcessor, Node)

private:
    struct Task {
        Callable callable;
        Callable callback;  // Called on main thread when complete
        std::atomic<bool> completed{false};
    };

    ThreadSafeQueue<std::shared_ptr<Task>> task_queue;
    std::vector<std::thread> worker_threads;
    std::atomic<bool> running{false};
    Vector<std::shared_ptr<Task>> completed_tasks;
    std::mutex completed_mutex;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("start_workers", "num_threads"),
                           &AsyncTaskProcessor::start_workers);
        ClassDB::bind_method(D_METHOD("submit_task", "task", "callback"),
                           &AsyncTaskProcessor::submit_task);
        ClassDB::bind_method(D_METHOD("process_completed"),
                           &AsyncTaskProcessor::process_completed);
    }

public:
    ~AsyncTaskProcessor() {
        stop_workers();
    }

    void start_workers(int num_threads) {
        if (running.load()) {
            return;  // Already running
        }

        running.store(true);
        worker_threads.reserve(num_threads);

        for (int i = 0; i < num_threads; i++) {
            worker_threads.emplace_back([this]() {
                worker_loop();
            });
        }
    }

    void stop_workers() {
        running.store(false);

        // Wake up all threads
        for (size_t i = 0; i < worker_threads.size(); i++) {
            task_queue.push(nullptr);  // Poison pill
        }

        for (auto& thread : worker_threads) {
            if (thread.joinable()) {
                thread.join();
            }
        }
        worker_threads.clear();
    }

    void submit_task(const Callable& task, const Callable& callback) {
        auto shared_task = std::make_shared<Task>();
        shared_task->callable = task;
        shared_task->callback = callback;

        task_queue.push(shared_task);
    }

    void process_completed() {
        // Called from main thread to process completed tasks
        std::lock_guard<std::mutex> lock(completed_mutex);

        for (auto& task : completed_tasks) {
            if (task->callback.is_valid()) {
                task->callback.call();
            }
        }
        completed_tasks.clear();
    }

private:
    void worker_loop() {
        while (running.load()) {
            auto task = task_queue.wait_and_pop();

            if (!task) {
                break;  // Poison pill received, exit thread
            }

            // Execute task
            if (task->callable.is_valid()) {
                task->callable.call();
            }

            // Mark as completed
            task->completed.store(true);

            // Add to completed list for main thread processing
            {
                std::lock_guard<std::mutex> lock(completed_mutex);
                completed_tasks.push_back(task);
            }
        }
    }
};
```

### Read-Write Locks

```cpp
class SharedResource : public RefCounted {
    GDCLASS(SharedResource, RefCounted)

private:
    mutable std::shared_mutex resource_mutex;
    Dictionary shared_data;
    HashMap<String, int> access_counts;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("read_value", "key"),
                           &SharedResource::read_value);
        ClassDB::bind_method(D_METHOD("write_value", "key", "value"),
                           &SharedResource::write_value);
        ClassDB::bind_method(D_METHOD("get_all_data"),
                           &SharedResource::get_all_data);
    }

public:
    Variant read_value(const String& key) {
        // Shared (read) lock - multiple readers allowed
        std::shared_lock<std::shared_mutex> lock(resource_mutex);

        // Track access for statistics
        access_counts[key]++;

        if (shared_data.has(key)) {
            return shared_data[key];
        }
        return Variant();
    }

    void write_value(const String& key, const Variant& value) {
        // Exclusive (write) lock - only one writer allowed
        std::unique_lock<std::shared_mutex> lock(resource_mutex);

        shared_data[key] = value;
    }

    Dictionary get_all_data() {
        // Shared lock for reading entire dictionary
        std::shared_lock<std::shared_mutex> lock(resource_mutex);
        return shared_data.duplicate();
    }

    void batch_update(const Dictionary& updates) {
        // Single write lock for multiple updates
        std::unique_lock<std::shared_mutex> lock(resource_mutex);

        for (const KeyValue<Variant, Variant>& E : updates) {
            shared_data[E.key] = E.value;
        }
    }

    HashMap<String, int> get_access_statistics() {
        std::shared_lock<std::shared_mutex> lock(resource_mutex);
        return access_counts;
    }
};
```

## Godot Object Thread Safety

### Main Thread Operations

```cpp
class MainThreadOperations : public RefCounted {
    GDCLASS(MainThreadOperations, RefCounted)

private:
    struct QueuedOperation {
        std::function<void()> operation;
        std::atomic<bool> completed{false};
        Variant result;
    };

    ThreadSafeQueue<std::shared_ptr<QueuedOperation>> main_thread_queue;
    static MainThreadOperations* instance;

protected:
    static void _bind_methods() {
        ClassDB::bind_static_method("MainThreadOperations",
                                   D_METHOD("get_instance"),
                                   &MainThreadOperations::get_instance);
        ClassDB::bind_method(D_METHOD("process_queue"),
                           &MainThreadOperations::process_queue);
    }

public:
    static MainThreadOperations* get_instance() {
        if (!instance) {
            instance = memnew(MainThreadOperations);
        }
        return instance;
    }

    template<typename Func>
    void queue_main_thread_operation(Func&& func) {
        auto operation = std::make_shared<QueuedOperation>();
        operation->operation = std::forward<Func>(func);
        main_thread_queue.push(operation);
    }

    template<typename Func>
    Variant call_on_main_thread_blocking(Func&& func) {
        if (ThreadContextManager::is_main_thread()) {
            // Already on main thread, execute directly
            return func();
        }

        auto operation = std::make_shared<QueuedOperation>();
        operation->operation = [&func, operation]() {
            operation->result = func();
            operation->completed.store(true);
        };

        main_thread_queue.push(operation);

        // Block until completed
        while (!operation->completed.load()) {
            std::this_thread::sleep_for(std::chrono::microseconds(100));
        }

        return operation->result;
    }

    int process_queue() {
        // Called from main thread (_process or similar)
        int processed = 0;
        std::shared_ptr<QueuedOperation> operation;

        while (main_thread_queue.try_pop(operation)) {
            if (operation && operation->operation) {
                operation->operation();
                processed++;
            }
        }

        return processed;
    }

    // Safe operations that can be called from any thread
    void safe_print(const String& message) {
        queue_main_thread_operation([message]() {
            UtilityFunctions::print(message);
        });
    }

    void safe_add_child(Node* parent, Node* child) {
        queue_main_thread_operation([parent, child]() {
            if (parent && child) {
                parent->add_child(child);
            }
        });
    }

    Vector2 safe_get_node_position(Node2D* node) {
        return call_on_main_thread_blocking([node]() -> Variant {
            if (node) {
                return node->get_position();
            }
            return Vector2();
        });
    }
};

MainThreadOperations* MainThreadOperations::instance = nullptr;
```

### Object Lifecycle Safety

```cpp
class SafeObjectReference : public RefCounted {
    GDCLASS(SafeObjectReference, RefCounted)

private:
    std::weak_ptr<Object> weak_ref;
    ObjectID object_id;
    mutable std::mutex reference_mutex;

public:
    SafeObjectReference(Object* obj) {
        if (obj) {
            object_id = obj->get_instance_id();
            // Note: This is a simplified example. In practice, you'd need
            // to integrate with Godot's object lifecycle system
        }
    }

    Object* get_object_safe() {
        std::lock_guard<std::mutex> lock(reference_mutex);

        if (object_id.is_valid()) {
            Object* obj = ObjectDB::get_instance(object_id);
            if (obj && obj->is_class("Object")) {
                return obj;
            }
        }

        return nullptr;
    }

    template<typename T>
    T* get_typed_object_safe() {
        Object* obj = get_object_safe();
        return Object::cast_to<T>(obj);
    }

    bool is_valid() {
        return get_object_safe() != nullptr;
    }

    ObjectID get_id() const {
        return object_id;
    }
};

class ThreadSafeNodeManager : public Node {
    GDCLASS(ThreadSafeNodeManager, Node)

private:
    HashMap<String, SafeObjectReference> managed_nodes;
    std::shared_mutex nodes_mutex;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("register_node", "name", "node"),
                           &ThreadSafeNodeManager::register_node);
        ClassDB::bind_method(D_METHOD("get_managed_node", "name"),
                           &ThreadSafeNodeManager::get_managed_node);
    }

public:
    void register_node(const String& name, Node* node) {
        std::unique_lock<std::shared_mutex> lock(nodes_mutex);

        if (node) {
            managed_nodes[name] = SafeObjectReference(node);
        }
    }

    Node* get_managed_node(const String& name) {
        std::shared_lock<std::shared_mutex> lock(nodes_mutex);

        auto it = managed_nodes.find(name);
        if (it != managed_nodes.end()) {
            return it->second.get_typed_object_safe<Node>();
        }
        return nullptr;
    }

    void cleanup_invalid_references() {
        std::unique_lock<std::shared_mutex> lock(nodes_mutex);

        Vector<String> to_remove;
        for (const KeyValue<String, SafeObjectReference>& E : managed_nodes) {
            if (!E.value.is_valid()) {
                to_remove.push_back(E.key);
            }
        }

        for (const String& key : to_remove) {
            managed_nodes.erase(key);
        }
    }
};
```

## Common Thread Safety Pitfalls

### Race Conditions

```cpp
class RaceConditionExample : public RefCounted {
    GDCLASS(RaceConditionExample, RefCounted)

private:
    int counter = 0;
    std::mutex counter_mutex;
    Vector<int> shared_vector;
    std::mutex vector_mutex;

public:
    // BAD: Race condition
    void unsafe_increment() {
        // Two threads can read the same value and write back the same increment
        counter = counter + 1;  // NOT THREAD SAFE
    }

    // GOOD: Protected increment
    void safe_increment() {
        std::lock_guard<std::mutex> lock(counter_mutex);
        counter = counter + 1;  // THREAD SAFE
    }

    // BAD: Race condition with containers
    void unsafe_vector_append(int value) {
        // Vector resize operations are not thread-safe
        shared_vector.append(value);  // NOT THREAD SAFE
    }

    // GOOD: Protected container operations
    void safe_vector_append(int value) {
        std::lock_guard<std::mutex> lock(vector_mutex);
        shared_vector.append(value);  // THREAD SAFE
    }

    // BAD: Check-then-act race condition
    int unsafe_pop_if_not_empty() {
        if (shared_vector.size() > 0) {  // Check
            // Another thread could empty the vector here!
            return shared_vector[shared_vector.size() - 1];  // Act - CRASH!
        }
        return -1;
    }

    // GOOD: Atomic check-then-act
    int safe_pop_if_not_empty() {
        std::lock_guard<std::mutex> lock(vector_mutex);
        if (shared_vector.size() > 0) {
            int value = shared_vector[shared_vector.size() - 1];
            shared_vector.remove_at(shared_vector.size() - 1);
            return value;
        }
        return -1;
    }
};
```

### Deadlocks

```cpp
class DeadlockPrevention : public RefCounted {
    GDCLASS(DeadlockPrevention, RefCounted)

private:
    std::mutex mutex_a;
    std::mutex mutex_b;
    int resource_a = 0;
    int resource_b = 0;

public:
    // BAD: Potential deadlock
    void transfer_a_to_b_unsafe(int amount) {
        std::lock_guard<std::mutex> lock_a(mutex_a);
        std::lock_guard<std::mutex> lock_b(mutex_b);  // Lock order: A then B

        resource_a -= amount;
        resource_b += amount;
    }

    void transfer_b_to_a_unsafe(int amount) {
        std::lock_guard<std::mutex> lock_b(mutex_b);
        std::lock_guard<std::mutex> lock_a(mutex_a);  // Lock order: B then A - DEADLOCK!

        resource_b -= amount;
        resource_a += amount;
    }

    // GOOD: Consistent lock ordering prevents deadlock
    void transfer_a_to_b_safe(int amount) {
        // Always lock in the same order
        std::lock_guard<std::mutex> lock_a(mutex_a);
        std::lock_guard<std::mutex> lock_b(mutex_b);

        resource_a -= amount;
        resource_b += amount;
    }

    void transfer_b_to_a_safe(int amount) {
        // Same lock order as above
        std::lock_guard<std::mutex> lock_a(mutex_a);
        std::lock_guard<std::mutex> lock_b(mutex_b);

        resource_b -= amount;
        resource_a += amount;
    }

    // BETTER: Use std::lock to acquire multiple mutexes atomically
    void atomic_transfer(int amount, bool a_to_b) {
        std::lock(mutex_a, mutex_b);
        std::lock_guard<std::mutex> lock_a(mutex_a, std::adopt_lock);
        std::lock_guard<std::mutex> lock_b(mutex_b, std::adopt_lock);

        if (a_to_b) {
            resource_a -= amount;
            resource_b += amount;
        } else {
            resource_b -= amount;
            resource_a += amount;
        }
    }
};
```

## Testing Thread Safety

### Thread Safety Validation

```cpp
class ThreadSafetyTester : public RefCounted {
    GDCLASS(ThreadSafetyTester, RefCounted)

private:
    std::atomic<int> test_counter{0};
    std::vector<std::thread> test_threads;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("run_concurrent_test", "num_threads", "operations_per_thread"),
                           &ThreadSafetyTester::run_concurrent_test);
    }

public:
    Dictionary run_concurrent_test(int num_threads, int operations_per_thread) {
        test_counter.store(0);
        std::atomic<int> completed_threads{0};
        auto start_time = std::chrono::high_resolution_clock::now();

        // Start threads
        test_threads.clear();
        test_threads.reserve(num_threads);

        for (int i = 0; i < num_threads; i++) {
            test_threads.emplace_back([this, operations_per_thread, &completed_threads]() {
                for (int j = 0; j < operations_per_thread; j++) {
                    test_counter.fetch_add(1, std::memory_order_relaxed);
                }
                completed_threads.fetch_add(1);
            });
        }

        // Wait for completion
        for (auto& thread : test_threads) {
            thread.join();
        }

        auto end_time = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end_time - start_time);

        Dictionary result;
        result["expected_count"] = num_threads * operations_per_thread;
        result["actual_count"] = test_counter.load();
        result["duration_us"] = static_cast<int>(duration.count());
        result["thread_safe"] = (test_counter.load() == num_threads * operations_per_thread);

        return result;
    }

    // Test with intentional race condition
    Dictionary run_unsafe_test(int num_threads, int operations_per_thread) {
        int unsafe_counter = 0;  // Not atomic, not protected
        std::vector<std::thread> threads;
        auto start_time = std::chrono::high_resolution_clock::now();

        for (int i = 0; i < num_threads; i++) {
            threads.emplace_back([&unsafe_counter, operations_per_thread]() {
                for (int j = 0; j < operations_per_thread; j++) {
                    unsafe_counter++;  // Race condition!
                }
            });
        }

        for (auto& thread : threads) {
            thread.join();
        }

        auto end_time = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end_time - start_time);

        Dictionary result;
        result["expected_count"] = num_threads * operations_per_thread;
        result["actual_count"] = unsafe_counter;
        result["duration_us"] = static_cast<int>(duration.count());
        result["thread_safe"] = (unsafe_counter == num_threads * operations_per_thread);

        return result;
    }
};
```

## Best Practices

### 1. Design for Thread Safety from the Start
```cpp
class ThreadSafeDesign : public RefCounted {
    GDCLASS(ThreadSafeDesign, RefCounted)

private:
    // Immutable data doesn't need synchronization
    const String config_name;
    const int max_items;

    // Mutable data with appropriate protection
    std::atomic<int> current_items{0};
    mutable std::shared_mutex items_mutex;
    Vector<String> items;

public:
    ThreadSafeDesign(const String& name, int max)
        : config_name(name), max_items(max) {}

    // Read-only operations are inherently thread-safe
    String get_config_name() const { return config_name; }
    int get_max_items() const { return max_items; }

    // Use appropriate synchronization for state changes
    bool add_item(const String& item) {
        if (current_items.load() >= max_items) {
            return false;
        }

        std::unique_lock<std::shared_mutex> lock(items_mutex);
        if (items.size() >= max_items) {
            return false;  // Double-check after acquiring lock
        }

        items.append(item);
        current_items.store(items.size());
        return true;
    }

    Vector<String> get_items() const {
        std::shared_lock<std::shared_mutex> lock(items_mutex);
        return items;  // Return copy for safety
    }
};
```

### 2. Minimize Locking Scope
```cpp
// BAD: Long-held locks
void bad_processing() {
    std::lock_guard<std::mutex> lock(data_mutex);

    // Expensive operation while holding lock
    for (int i = 0; i < 1000000; i++) {
        process_item(i);
    }

    // Update shared data
    shared_result = final_result;
}

// GOOD: Minimal lock scope
void good_processing() {
    // Do expensive work outside of lock
    int result = 0;
    for (int i = 0; i < 1000000; i++) {
        result += process_item(i);
    }

    // Quick update with lock
    {
        std::lock_guard<std::mutex> lock(data_mutex);
        shared_result = result;
    }
}
```

### 3. Use RAII for Exception Safety
```cpp
class RAIILocking : public RefCounted {
private:
    std::mutex resource_mutex;
    std::unique_ptr<Resource> resource;

public:
    void safe_operation() {
        std::lock_guard<std::mutex> lock(resource_mutex);

        // Even if this throws an exception, lock is automatically released
        resource->risky_operation();

        // Lock automatically released when scope exits
    }

    // Custom RAII helper for more complex scenarios
    class ResourceGuard {
        RAIILocking* owner;

    public:
        ResourceGuard(RAIILocking* p_owner) : owner(p_owner) {
            owner->resource_mutex.lock();
        }

        ~ResourceGuard() {
            owner->resource_mutex.unlock();
        }

        Resource* get() {
            return owner->resource.get();
        }
    };

    void complex_operation() {
        ResourceGuard guard(this);

        // Multiple operations with automatic cleanup
        guard.get()->operation1();
        guard.get()->operation2();

        // Lock automatically released
    }
};
```

## Conclusion

Thread safety in GDExtension requires careful consideration of:

1. **Godot's threading model** - Understanding which threads call your code
2. **Appropriate synchronization primitives** - Mutexes, atomics, lock-free structures
3. **Object lifecycle management** - Safe references across threads
4. **Main thread operations** - Queueing operations that must run on main thread
5. **Common pitfalls** - Race conditions, deadlocks, and memory ordering

Always design for thread safety from the beginning, use appropriate synchronization mechanisms, and test thoroughly with concurrent workloads. When in doubt, err on the side of caution and use proper locking rather than risking data corruption or crashes.
