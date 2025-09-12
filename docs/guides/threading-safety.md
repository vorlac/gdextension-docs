# Thread Safety - Concurrent Programming Guide

## Introduction

**Safe concurrent programming in GDExtension:** Threading safety is critical when building GDExtensions that use multiple threads or interact with Godot's threaded subsystems. This guide covers thread-safe patterns, synchronization techniques, and best practices for avoiding race conditions and deadlocks.

## Understanding Godot's Threading Context

**When and where your GDExtension code runs:** Godot uses multiple threads for different purposes, and understanding which thread context your code executes in is essential for thread safety.

### Thread Contexts in Godot

```cpp
class ThreadContextExample : public Node {
    GDCLASS(ThreadContextExample, Node)

private:
    std::thread::id main_thread_id;
    std::atomic<int> call_count{0};
    std::mutex log_mutex;

public:
    ThreadContextExample() {
        main_thread_id = std::this_thread::get_id();
    }

    void _ready() override {
        log_thread_context("_ready called");

        // Main thread operations are safe
        modify_scene_tree();
    }

    void _process(double delta) override {
        log_thread_context("_process called");

        // This runs on the main thread - safe for most operations
        update_ui_elements();
    }

    void _physics_process(double delta) override {
        log_thread_context("_physics_process called");

        // This might run on a physics thread - be careful!
        update_physics_state();
    }

    // Called from audio thread
    virtual void _mix_audio(AudioFrame* buffer, int frame_count) {
        log_thread_context("_mix_audio called");

        // Audio thread - very restrictive!
        // No Godot API calls, minimal allocations
        process_audio_safely(buffer, frame_count);
    }

private:
    void log_thread_context(const String& context) {
        std::lock_guard<std::mutex> lock(log_mutex);

        std::thread::id current_id = std::this_thread::get_id();
        bool is_main_thread = (current_id == main_thread_id);

        call_count.fetch_add(1);

        // Safe to print from main thread, be careful from others
        if (is_main_thread) {
            UtilityFunctions::print(context, "- Main thread, call", call_count.load());
        }
    }

    void modify_scene_tree() {
        // Scene tree operations - MAIN THREAD ONLY
        Node* child = memnew(Node);
        child->set_name("ThreadSafeChild");
        add_child(child);
    }

    void update_ui_elements() {
        // UI updates - MAIN THREAD ONLY
        Control* ui = Object::cast_to<Control>(get_node_or_null(NodePath("UI")));
        if (ui) {
            // ui->set_size(Vector2(100, 100));  // Safe on main thread
        }
    }

    void update_physics_state() {
        // Physics state - potentially physics thread
        // Use atomic operations or proper synchronization
        call_count.fetch_add(1);
    }

    void process_audio_safely(AudioFrame* buffer, int frame_count) {
        // Audio processing - audio thread
        // NO Godot API calls, NO memory allocation, NO complex operations

        for (int i = 0; i < frame_count; i++) {
            // Simple audio processing only
            buffer[i].l *= 0.5f;
            buffer[i].r *= 0.5f;
        }
    }
};
```

### Thread-Safe Communication Patterns

```cpp
class ThreadSafeCommunication : public Node {
    GDCLASS(ThreadSafeCommunication, Node)

private:
    // Thread-safe data structures
    std::atomic<bool> worker_running{false};
    std::thread worker_thread;

    // Message queue for thread communication
    std::queue<String> message_queue;
    std::mutex queue_mutex;
    std::condition_variable queue_cv;

    // Main thread command queue
    std::queue<std::function<void()>> main_thread_commands;
    std::mutex command_mutex;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("start_worker"), &ThreadSafeCommunication::start_worker);
        ClassDB::bind_method(D_METHOD("stop_worker"), &ThreadSafeCommunication::stop_worker);
        ClassDB::bind_method(D_METHOD("send_message_to_worker", "message"),
                           &ThreadSafeCommunication::send_message_to_worker);
        ClassDB::bind_method(D_METHOD("process_main_thread_commands"),
                           &ThreadSafeCommunication::process_main_thread_commands);
    }

public:
    ~ThreadSafeCommunication() {
        stop_worker();
    }

    void start_worker() {
        if (worker_running.load()) {
            return;  // Already running
        }

        worker_running.store(true);
        worker_thread = std::thread([this]() {
            worker_loop();
        });
    }

    void stop_worker() {
        worker_running.store(false);

        // Wake up worker thread
        queue_cv.notify_all();

        if (worker_thread.joinable()) {
            worker_thread.join();
        }
    }

    void send_message_to_worker(const String& message) {
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            message_queue.push(message);
        }
        queue_cv.notify_one();
    }

    void _process(double delta) override {
        // Process commands that need to run on main thread
        process_main_thread_commands();
    }

    void process_main_thread_commands() {
        std::lock_guard<std::mutex> lock(command_mutex);

        while (!main_thread_commands.empty()) {
            auto command = main_thread_commands.front();
            main_thread_commands.pop();

            // Execute command on main thread
            command();
        }
    }

private:
    void worker_loop() {
        while (worker_running.load()) {
            String message;

            // Wait for message
            {
                std::unique_lock<std::mutex> lock(queue_mutex);
                queue_cv.wait(lock, [this] {
                    return !message_queue.empty() || !worker_running.load();
                });

                if (!worker_running.load()) {
                    break;
                }

                if (!message_queue.empty()) {
                    message = message_queue.front();
                    message_queue.pop();
                }
            }

            if (!message.is_empty()) {
                process_worker_message(message);
            }
        }
    }

    void process_worker_message(const String& message) {
        // Worker thread processing
        std::this_thread::sleep_for(std::chrono::milliseconds(100));  // Simulate work

        // Send result back to main thread
        queue_main_thread_command([this, message]() {
            // This will execute on main thread
            UtilityFunctions::print("Worker processed:", message);

            // Safe to call Godot APIs here
            Node* result_node = memnew(Node);
            result_node->set_name("WorkerResult_" + message);
            add_child(result_node);
        });
    }

    void queue_main_thread_command(std::function<void()> command) {
        std::lock_guard<std::mutex> lock(command_mutex);
        main_thread_commands.push(command);
    }
};
```

## Synchronization Primitives

### Atomic Operations

```cpp
class AtomicOperationsExample : public RefCounted {
    GDCLASS(AtomicOperationsExample, RefCounted)

private:
    // Atomic primitives for lock-free operations
    std::atomic<int> counter{0};
    std::atomic<float> average_value{0.0f};
    std::atomic<bool> is_processing{false};
    std::atomic<uint64_t> total_bytes_processed{0};

    // Atomic pointers
    std::atomic<Node*> current_target{nullptr};

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("increment_counter"), &AtomicOperationsExample::increment_counter);
        ClassDB::bind_method(D_METHOD("get_counter"), &AtomicOperationsExample::get_counter);
        ClassDB::bind_method(D_METHOD("update_average", "new_value"), &AtomicOperationsExample::update_average);
        ClassDB::bind_method(D_METHOD("try_start_processing"), &AtomicOperationsExample::try_start_processing);
    }

public:
    void increment_counter() {
        // Thread-safe increment
        counter.fetch_add(1, std::memory_order_relaxed);
    }

    int get_counter() const {
        return counter.load(std::memory_order_relaxed);
    }

    void update_average(float new_value) {
        // Lock-free update using compare-and-swap
        float expected = average_value.load();
        float new_avg;

        do {
            new_avg = (expected + new_value) / 2.0f;
        } while (!average_value.compare_exchange_weak(expected, new_avg,
                                                     std::memory_order_release,
                                                     std::memory_order_relaxed));
    }

    bool try_start_processing() {
        // Atomic test-and-set pattern
        bool expected = false;
        return is_processing.compare_exchange_strong(expected, true,
                                                   std::memory_order_acquire);
    }

    void finish_processing() {
        is_processing.store(false, std::memory_order_release);
    }

    void add_bytes_processed(uint64_t bytes) {
        total_bytes_processed.fetch_add(bytes, std::memory_order_relaxed);
    }

    uint64_t get_total_bytes_processed() const {
        return total_bytes_processed.load(std::memory_order_relaxed);
    }

    void set_target(Node* target) {
        current_target.store(target, std::memory_order_release);
    }

    Node* get_target() const {
        return current_target.load(std::memory_order_acquire);
    }

    // Atomic operations with memory ordering
    void demonstrate_memory_ordering() {
        // Relaxed ordering - no synchronization, only atomicity
        counter.fetch_add(1, std::memory_order_relaxed);

        // Acquire-release ordering - synchronizes with paired operations
        is_processing.store(true, std::memory_order_release);  // Release
        bool processing = is_processing.load(std::memory_order_acquire);  // Acquire

        // Sequential consistency - strongest ordering (default)
        counter.fetch_add(1);  // Equivalent to memory_order_seq_cst
    }
};
```

### Mutex and Lock Patterns

```cpp
class MutexPatterns : public RefCounted {
    GDCLASS(MutexPatterns, RefCounted)

private:
    // Different mutex types for different needs
    std::mutex exclusive_mutex;
    std::recursive_mutex recursive_mutex;
    std::shared_mutex shared_mutex;
    std::timed_mutex timed_mutex;

    // Protected data
    Vector<String> shared_list;
    HashMap<String, int> shared_map;
    int recursive_counter = 0;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("add_item", "item"), &MutexPatterns::add_item);
        ClassDB::bind_method(D_METHOD("get_items"), &MutexPatterns::get_items);
        ClassDB::bind_method(D_METHOD("update_map", "key", "value"), &MutexPatterns::update_map);
        ClassDB::bind_method(D_METHOD("recursive_operation", "depth"), &MutexPatterns::recursive_operation);
    }

public:
    void add_item(const String& item) {
        // Exclusive access for writing
        std::lock_guard<std::mutex> lock(exclusive_mutex);
        shared_list.append(item);

        UtilityFunctions::print("Added item:", item, "Total:", shared_list.size());
    }

    Array get_items() {
        // Shared access for reading
        std::shared_lock<std::shared_mutex> lock(shared_mutex);

        Array result;
        for (const String& item : shared_list) {
            result.append(item);
        }
        return result;
    }

    void update_map(const String& key, int value) {
        // Try to acquire lock with timeout
        if (timed_mutex.try_lock_for(std::chrono::milliseconds(100))) {
            shared_map[key] = value;
            timed_mutex.unlock();
            UtilityFunctions::print("Updated map:", key, "=", value);
        } else {
            WARN_PRINT("Failed to acquire lock for map update");
        }
    }

    void recursive_operation(int depth) {
        // Recursive mutex allows same thread to lock multiple times
        std::lock_guard<std::recursive_mutex> lock(recursive_mutex);

        recursive_counter++;
        UtilityFunctions::print("Recursive operation depth:", depth, "counter:", recursive_counter);

        if (depth > 0) {
            recursive_operation(depth - 1);  // Recursive call with same lock
        }
    }

    // RAII lock management
    void demonstrate_raii_locking() {
        {
            std::lock_guard<std::mutex> auto_lock(exclusive_mutex);
            // Lock is automatically acquired

            // Do work...
            shared_list.append("RAII example");

            // Lock is automatically released when auto_lock goes out of scope
        }

        // Manual lock management (not recommended)
        exclusive_mutex.lock();
        try {
            shared_list.append("Manual locking");
            exclusive_mutex.unlock();
        } catch (...) {
            exclusive_mutex.unlock();  // Must unlock in exception case
            throw;
        }
    }

    // Multiple lock acquisition (deadlock prevention)
    void transfer_data(MutexPatterns* other) {
        if (!other) return;

        // Use std::lock to acquire multiple mutexes atomically
        std::lock(exclusive_mutex, other->exclusive_mutex);

        // Adopt the already-acquired locks
        std::lock_guard<std::mutex> lock1(exclusive_mutex, std::adopt_lock);
        std::lock_guard<std::mutex> lock2(other->exclusive_mutex, std::adopt_lock);

        // Safe to access both objects' data
        String item = shared_list.is_empty() ? "" : shared_list[0];
        if (!item.is_empty()) {
            other->shared_list.append(item);
            shared_list.remove_at(0);
        }
    }

    // Conditional locking
    bool try_quick_operation() {
        if (exclusive_mutex.try_lock()) {
            // Got the lock, do quick operation
            shared_list.append("Quick operation");
            exclusive_mutex.unlock();
            return true;
        } else {
            // Couldn't get lock, skip operation
            return false;
        }
    }

    // Scoped locking with custom RAII
    class ScopedOperationLock {
        MutexPatterns* owner;

    public:
        ScopedOperationLock(MutexPatterns* p_owner) : owner(p_owner) {
            owner->exclusive_mutex.lock();
            UtilityFunctions::print("Operation started");
        }

        ~ScopedOperationLock() {
            owner->exclusive_mutex.unlock();
            UtilityFunctions::print("Operation completed");
        }
    };

    void complex_operation() {
        ScopedOperationLock operation_lock(this);

        // Do complex work with automatic cleanup
        shared_list.append("Complex operation result");

        // Lock is automatically released when operation_lock goes out of scope
    }
};
```

## Lock-Free Data Structures

### Lock-Free Queue

```cpp
template<typename T, size_t Size>
class LockFreeQueue {
private:
    struct alignas(64) Node {  // Cache line alignment
        std::atomic<T> data;
        std::atomic<bool> ready{false};
    };

    alignas(64) std::atomic<size_t> write_index{0};
    alignas(64) std::atomic<size_t> read_index{0};
    Node nodes[Size];

public:
    bool try_enqueue(const T& item) {
        size_t write_pos = write_index.load(std::memory_order_relaxed);
        size_t next_write = (write_pos + 1) % Size;

        // Check if queue is full
        if (next_write == read_index.load(std::memory_order_acquire)) {
            return false;
        }

        // Store the data
        nodes[write_pos].data.store(item, std::memory_order_relaxed);
        nodes[write_pos].ready.store(true, std::memory_order_release);

        // Update write index
        write_index.store(next_write, std::memory_order_release);
        return true;
    }

    bool try_dequeue(T& item) {
        size_t read_pos = read_index.load(std::memory_order_relaxed);

        // Check if queue is empty
        if (read_pos == write_index.load(std::memory_order_acquire)) {
            return false;
        }

        // Wait for data to be ready
        while (!nodes[read_pos].ready.load(std::memory_order_acquire)) {
            std::this_thread::yield();
        }

        // Get the data
        item = nodes[read_pos].data.load(std::memory_order_relaxed);
        nodes[read_pos].ready.store(false, std::memory_order_relaxed);

        // Update read index
        read_index.store((read_pos + 1) % Size, std::memory_order_release);
        return true;
    }

    size_t size() const {
        size_t write_pos = write_index.load(std::memory_order_acquire);
        size_t read_pos = read_index.load(std::memory_order_acquire);

        if (write_pos >= read_pos) {
            return write_pos - read_pos;
        } else {
            return Size - read_pos + write_pos;
        }
    }

    bool empty() const {
        return read_index.load(std::memory_order_acquire) ==
               write_index.load(std::memory_order_acquire);
    }

    bool full() const {
        size_t write_pos = write_index.load(std::memory_order_acquire);
        size_t next_write = (write_pos + 1) % Size;
        return next_write == read_index.load(std::memory_order_acquire);
    }
};

class LockFreeExample : public Node {
    GDCLASS(LockFreeExample, Node)

private:
    LockFreeQueue<String, 1024> message_queue;
    std::thread producer_thread;
    std::thread consumer_thread;
    std::atomic<bool> running{false};

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("start_lock_free_test"),
                           &LockFreeExample::start_lock_free_test);
        ClassDB::bind_method(D_METHOD("stop_lock_free_test"),
                           &LockFreeExample::stop_lock_free_test);
        ClassDB::bind_method(D_METHOD("send_message", "message"),
                           &LockFreeExample::send_message);
    }

public:
    ~LockFreeExample() {
        stop_lock_free_test();
    }

    void start_lock_free_test() {
        if (running.load()) return;

        running.store(true);

        // Start producer thread
        producer_thread = std::thread([this]() {
            int counter = 0;
            while (running.load()) {
                String message = "Auto message " + itos(counter++);

                if (!message_queue.try_enqueue(message)) {
                    // Queue full, wait a bit
                    std::this_thread::sleep_for(std::chrono::milliseconds(1));
                } else {
                    std::this_thread::sleep_for(std::chrono::milliseconds(10));
                }
            }
        });

        // Start consumer thread
        consumer_thread = std::thread([this]() {
            String message;
            while (running.load()) {
                if (message_queue.try_dequeue(message)) {
                    // Process message (simulate work)
                    std::this_thread::sleep_for(std::chrono::milliseconds(5));

                    // Queue result for main thread
                    queue_main_thread_print(message);
                } else {
                    std::this_thread::yield();
                }
            }
        });
    }

    void stop_lock_free_test() {
        running.store(false);

        if (producer_thread.joinable()) {
            producer_thread.join();
        }

        if (consumer_thread.joinable()) {
            consumer_thread.join();
        }
    }

    void send_message(const String& message) {
        if (!message_queue.try_enqueue(message)) {
            WARN_PRINT("Message queue full, dropping message: " + message);
        }
    }

    void _process(double delta) override {
        // Process queued print messages on main thread
        process_main_thread_prints();
    }

private:
    LockFreeQueue<String, 256> print_queue;

    void queue_main_thread_print(const String& message) {
        print_queue.try_enqueue("Processed: " + message);
    }

    void process_main_thread_prints() {
        String message;
        int processed = 0;

        // Limit processing to avoid frame drops
        while (processed < 10 && print_queue.try_dequeue(message)) {
            UtilityFunctions::print(message);
            processed++;
        }
    }
};
```

## Thread-Safe Singletons

### Thread-Safe Singleton Pattern

```cpp
class ThreadSafeSingleton : public RefCounted {
    GDCLASS(ThreadSafeSingleton, RefCounted)

private:
    static std::once_flag initialized_flag;
    static ThreadSafeSingleton* instance;
    static std::mutex instance_mutex;

    // Thread-safe data
    std::atomic<int> request_count{0};
    mutable std::shared_mutex data_mutex;
    HashMap<String, Variant> shared_data;

protected:
    static void _bind_methods() {
        ClassDB::bind_static_method("ThreadSafeSingleton",
                                   D_METHOD("get_instance"),
                                   &ThreadSafeSingleton::get_instance);
        ClassDB::bind_method(D_METHOD("increment_requests"),
                           &ThreadSafeSingleton::increment_requests);
        ClassDB::bind_method(D_METHOD("get_request_count"),
                           &ThreadSafeSingleton::get_request_count);
        ClassDB::bind_method(D_METHOD("set_data", "key", "value"),
                           &ThreadSafeSingleton::set_data);
        ClassDB::bind_method(D_METHOD("get_data", "key"),
                           &ThreadSafeSingleton::get_data);
    }

    ThreadSafeSingleton() {
        // Private constructor
        UtilityFunctions::print("ThreadSafeSingleton instance created");
    }

public:
    static ThreadSafeSingleton* get_instance() {
        // Thread-safe lazy initialization using std::call_once
        std::call_once(initialized_flag, []() {
            instance = memnew(ThreadSafeSingleton);
        });
        return instance;
    }

    void increment_requests() {
        request_count.fetch_add(1, std::memory_order_relaxed);
    }

    int get_request_count() const {
        return request_count.load(std::memory_order_relaxed);
    }

    void set_data(const String& key, const Variant& value) {
        std::unique_lock<std::shared_mutex> lock(data_mutex);
        shared_data[key] = value;
    }

    Variant get_data(const String& key) const {
        std::shared_lock<std::shared_mutex> lock(data_mutex);

        if (shared_data.has(key)) {
            return shared_data[key];
        }
        return Variant();
    }

    // Bulk operations for better performance
    void set_data_batch(const Dictionary& data) {
        std::unique_lock<std::shared_mutex> lock(data_mutex);

        for (const KeyValue<Variant, Variant>& E : data) {
            shared_data[E.key] = E.value;
        }
    }

    Dictionary get_all_data() const {
        std::shared_lock<std::shared_mutex> lock(data_mutex);

        Dictionary result;
        for (const KeyValue<String, Variant>& E : shared_data) {
            result[E.key] = E.value;
        }
        return result;
    }

    static void cleanup() {
        std::lock_guard<std::mutex> lock(instance_mutex);
        if (instance) {
            memdelete(instance);
            instance = nullptr;
        }
    }
};

// Static member definitions
std::once_flag ThreadSafeSingleton::initialized_flag;
ThreadSafeSingleton* ThreadSafeSingleton::instance = nullptr;
std::mutex ThreadSafeSingleton::instance_mutex;
```

## Deadlock Prevention

### Deadlock Prevention Strategies

```cpp
class DeadlockPrevention : public RefCounted {
    GDCLASS(DeadlockPrevention, RefCounted)

private:
    std::mutex resource_a_mutex;
    std::mutex resource_b_mutex;
    std::mutex resource_c_mutex;

    int resource_a = 100;
    int resource_b = 200;
    int resource_c = 300;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("transfer_a_to_b", "amount"),
                           &DeadlockPrevention::transfer_a_to_b);
        ClassDB::bind_method(D_METHOD("transfer_b_to_a", "amount"),
                           &DeadlockPrevention::transfer_b_to_a);
        ClassDB::bind_method(D_METHOD("three_way_transfer", "amount"),
                           &DeadlockPrevention::three_way_transfer);
    }

public:
    // BAD: Inconsistent lock ordering can cause deadlocks
    void transfer_a_to_b_unsafe(int amount) {
        std::lock_guard<std::mutex> lock_a(resource_a_mutex);  // A first
        std::lock_guard<std::mutex> lock_b(resource_b_mutex);  // B second

        resource_a -= amount;
        resource_b += amount;
    }

    void transfer_b_to_a_unsafe(int amount) {
        std::lock_guard<std::mutex> lock_b(resource_b_mutex);  // B first - DEADLOCK!
        std::lock_guard<std::mutex> lock_a(resource_a_mutex);  // A second

        resource_b -= amount;
        resource_a += amount;
    }

    // GOOD: Consistent lock ordering prevents deadlocks
    void transfer_a_to_b(int amount) {
        // Always lock in the same order: A before B
        std::lock_guard<std::mutex> lock_a(resource_a_mutex);
        std::lock_guard<std::mutex> lock_b(resource_b_mutex);

        resource_a -= amount;
        resource_b += amount;
    }

    void transfer_b_to_a(int amount) {
        // Same lock order: A before B
        std::lock_guard<std::mutex> lock_a(resource_a_mutex);
        std::lock_guard<std::mutex> lock_b(resource_b_mutex);

        resource_b -= amount;
        resource_a += amount;
    }

    // BETTER: Use std::lock for atomic multi-mutex acquisition
    void three_way_transfer(int amount) {
        // Acquire all three mutexes atomically
        std::lock(resource_a_mutex, resource_b_mutex, resource_c_mutex);

        // Use adopt_lock since mutexes are already locked
        std::lock_guard<std::mutex> lock_a(resource_a_mutex, std::adopt_lock);
        std::lock_guard<std::mutex> lock_b(resource_b_mutex, std::adopt_lock);
        std::lock_guard<std::mutex> lock_c(resource_c_mutex, std::adopt_lock);

        // Perform three-way transfer
        resource_a -= amount;
        resource_b += amount / 2;
        resource_c += amount / 2;
    }

    // Timeout-based deadlock avoidance
    bool try_transfer_with_timeout(int amount, std::chrono::milliseconds timeout) {
        std::unique_lock<std::mutex> lock_a(resource_a_mutex, std::defer_lock);
        std::unique_lock<std::mutex> lock_b(resource_b_mutex, std::defer_lock);

        // Try to lock both within timeout
        if (std::try_lock(lock_a, lock_b) == -1) {
            // Success - both locks acquired
            resource_a -= amount;
            resource_b += amount;
            return true;
        } else {
            // Failed to acquire both locks
            return false;
        }
    }

    // Hierarchical locking (by address)
    void hierarchical_transfer(DeadlockPrevention* other, int amount) {
        if (!other) return;

        // Lock mutexes in address order to prevent deadlocks
        std::mutex* first_mutex;
        std::mutex* second_mutex;

        if (this < other) {
            first_mutex = &resource_a_mutex;
            second_mutex = &other->resource_a_mutex;
        } else {
            first_mutex = &other->resource_a_mutex;
            second_mutex = &resource_a_mutex;
        }

        std::lock_guard<std::mutex> lock1(*first_mutex);
        std::lock_guard<std::mutex> lock2(*second_mutex);

        // Safe to transfer between objects
        resource_a -= amount;
        other->resource_a += amount;
    }

    Dictionary get_resources() const {
        // Use std::lock for reading multiple resources atomically
        std::lock(resource_a_mutex, resource_b_mutex, resource_c_mutex);
        std::lock_guard<std::mutex> lock_a(resource_a_mutex, std::adopt_lock);
        std::lock_guard<std::mutex> lock_b(resource_b_mutex, std::adopt_lock);
        std::lock_guard<std::mutex> lock_c(resource_c_mutex, std::adopt_lock);

        Dictionary result;
        result["a"] = resource_a;
        result["b"] = resource_b;
        result["c"] = resource_c;
        return result;
    }
};
```

## Testing Thread Safety

### Thread Safety Testing Framework

```cpp
class ThreadSafetyTester : public RefCounted {
    GDCLASS(ThreadSafetyTester, RefCounted)

private:
    std::atomic<int> test_counter{0};
    std::atomic<bool> test_failed{false};
    std::atomic<int> active_threads{0};

    Vector<std::thread> test_threads;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("run_concurrency_test", "thread_count", "iterations"),
                           &ThreadSafetyTester::run_concurrency_test);
        ClassDB::bind_method(D_METHOD("run_atomic_test"), &ThreadSafetyTester::run_atomic_test);
        ClassDB::bind_method(D_METHOD("run_mutex_test"), &ThreadSafetyTester::run_mutex_test);
    }

public:
    Dictionary run_concurrency_test(int thread_count, int iterations) {
        test_counter.store(0);
        test_failed.store(false);
        active_threads.store(0);

        auto start_time = std::chrono::high_resolution_clock::now();

        // Start test threads
        test_threads.clear();
        test_threads.reserve(thread_count);

        for (int i = 0; i < thread_count; i++) {
            test_threads.emplace_back([this, iterations]() {
                active_threads.fetch_add(1);

                for (int j = 0; j < iterations; j++) {
                    // Test atomic increment
                    int old_value = test_counter.fetch_add(1);

                    // Verify no race condition occurred
                    if (old_value < 0) {
                        test_failed.store(true);
                        break;
                    }

                    // Simulate some work
                    std::this_thread::yield();
                }

                active_threads.fetch_sub(1);
            });
        }

        // Wait for all threads to complete
        for (auto& thread : test_threads) {
            thread.join();
        }

        auto end_time = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end_time - start_time);

        Dictionary result;
        result["expected_count"] = thread_count * iterations;
        result["actual_count"] = test_counter.load();
        result["test_passed"] = !test_failed.load() && (test_counter.load() == thread_count * iterations);
        result["duration_microseconds"] = static_cast<int>(duration.count());
        result["operations_per_second"] = static_cast<int>((thread_count * iterations * 1000000.0) / duration.count());

        return result;
    }

    Dictionary run_atomic_test() {
        const int num_threads = 8;
        const int iterations = 100000;

        std::atomic<int> atomic_counter{0};
        int unsafe_counter = 0;
        std::mutex unsafe_mutex;

        Vector<std::thread> threads;
        auto start_time = std::chrono::high_resolution_clock::now();

        // Test atomic operations
        for (int i = 0; i < num_threads; i++) {
            threads.emplace_back([&atomic_counter, &unsafe_counter, &unsafe_mutex, iterations]() {
                for (int j = 0; j < iterations; j++) {
                    // Atomic increment (thread-safe)
                    atomic_counter.fetch_add(1, std::memory_order_relaxed);

                    // Unsafe increment for comparison
                    {
                        std::lock_guard<std::mutex> lock(unsafe_mutex);
                        unsafe_counter++;
                    }
                }
            });
        }

        for (auto& thread : threads) {
            thread.join();
        }

        auto end_time = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end_time - start_time);

        Dictionary result;
        result["atomic_result"] = atomic_counter.load();
        result["mutex_result"] = unsafe_counter;
        result["expected_result"] = num_threads * iterations;
        result["atomic_correct"] = (atomic_counter.load() == num_threads * iterations);
        result["mutex_correct"] = (unsafe_counter == num_threads * iterations);
        result["duration_microseconds"] = static_cast<int>(duration.count());

        return result;
    }

    Dictionary run_mutex_test() {
        const int num_threads = 4;
        const int iterations = 50000;

        std::mutex test_mutex;
        int shared_counter = 0;
        bool data_corruption = false;

        Vector<std::thread> threads;
        auto start_time = std::chrono::high_resolution_clock::now();

        for (int i = 0; i < num_threads; i++) {
            threads.emplace_back([&test_mutex, &shared_counter, &data_corruption, iterations]() {
                for (int j = 0; j < iterations; j++) {
                    std::lock_guard<std::mutex> lock(test_mutex);

                    int old_value = shared_counter;
                    std::this_thread::yield();  // Increase chance of race condition without mutex
                    shared_counter = old_value + 1;

                    // Verify integrity
                    if (shared_counter <= old_value) {
                        data_corruption = true;
                    }
                }
            });
        }

        for (auto& thread : threads) {
            thread.join();
        }

        auto end_time = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end_time - start_time);

        Dictionary result;
        result["final_count"] = shared_counter;
        result["expected_count"] = num_threads * iterations;
        result["test_passed"] = !data_corruption && (shared_counter == num_threads * iterations);
        result["data_corruption_detected"] = data_corruption;
        result["duration_microseconds"] = static_cast<int>(duration.count());

        return result;
    }
};
```

## Best Practices Summary

### 1. Thread Context Awareness
- Know which thread context your code runs in
- Use main thread for Godot API calls
- Minimize work in audio/physics threads

### 2. Synchronization Strategy
- Prefer atomic operations for simple data
- Use mutexes for complex critical sections
- Consider lock-free structures for high performance

### 3. Deadlock Prevention
- Always acquire locks in the same order
- Use std::lock for multiple mutex acquisition
- Implement timeout-based locking where appropriate

### 4. Testing and Validation
- Test with multiple threads under load
- Use thread sanitizers during development
- Validate assumptions with stress tests

## Conclusion

Thread safety in GDExtension requires:
- **Understanding Godot's threading model**
- **Appropriate synchronization primitives**
- **Deadlock prevention strategies**
- **Performance-conscious design**
- **Comprehensive testing**

Proper thread safety ensures stable, reliable extensions that can leverage multi-core performance without risking data corruption or crashes.
