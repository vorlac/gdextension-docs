# Performance Optimization Techniques

## Introduction

**Maximizing performance in GDExtension:** This guide covers advanced techniques for optimizing GDExtension performance, from minimizing cross-boundary calls to implementing cache-friendly data structures. Understanding these patterns is crucial for building high-performance extensions that can handle demanding real-time applications.

## Cross-Boundary Call Optimization

**Minimizing the cost of engine communication:** Every call across the GDExtension boundary has overhead. Reducing and optimizing these calls is often the biggest performance win.

### Batch Operations

```cpp
class BatchProcessor : public RefCounted {
    GDCLASS(BatchProcessor, RefCounted)

private:
    Vector<Vector3> pending_positions;
    Vector<Node3D*> target_nodes;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("add_position_update", "node", "position"),
                           &BatchProcessor::add_position_update);
        ClassDB::bind_method(D_METHOD("flush_updates"),
                           &BatchProcessor::flush_updates);
    }

public:
    // Bad: Individual cross-boundary calls
    void update_position_immediate(Node3D *node, const Vector3 &pos) {
        node->set_position(pos);  // Cross-boundary call per update
    }

    // Good: Batch updates
    void add_position_update(Node3D *node, const Vector3 &pos) {
        target_nodes.append(node);
        pending_positions.append(pos);
    }

    void flush_updates() {
        // Single batch of cross-boundary calls
        for (int i = 0; i < target_nodes.size(); i++) {
            target_nodes[i]->set_position(pending_positions[i]);
        }
        target_nodes.clear();
        pending_positions.clear();
    }
};
```

### Cache Engine Data

```cpp
class CachedNodeManager : public Node {
    GDCLASS(CachedNodeManager, Node)

private:
    struct NodeCache {
        Node *node;
        Vector3 last_position;
        Vector3 last_scale;
        bool position_dirty = false;
        bool scale_dirty = false;
    };

    Vector<NodeCache> cached_nodes;
    bool batch_updates = true;

public:
    void cache_node(Node *node) {
        NodeCache cache;
        cache.node = node;

        // Cache current values to avoid repeated queries
        if (Node3D *node3d = Object::cast_to<Node3D>(node)) {
            cache.last_position = node3d->get_position();
            cache.last_scale = node3d->get_scale();
        }

        cached_nodes.append(cache);
    }

    void update_cached_position(int index, const Vector3 &new_pos) {
        ERR_FAIL_INDEX(index, cached_nodes.size());

        NodeCache &cache = cached_nodes.write[index];
        if (cache.last_position.distance_squared_to(new_pos) > 0.001f) {
            cache.last_position = new_pos;
            cache.position_dirty = true;
        }
    }

    void flush_dirty_updates() {
        for (NodeCache &cache : cached_nodes) {
            Node3D *node3d = Object::cast_to<Node3D>(cache.node);
            if (!node3d) continue;

            if (cache.position_dirty) {
                node3d->set_position(cache.last_position);
                cache.position_dirty = false;
            }

            if (cache.scale_dirty) {
                node3d->set_scale(cache.last_scale);
                cache.scale_dirty = false;
            }
        }
    }
};
```

### Reduce Method Call Overhead

```cpp
class OptimizedCalculator : public RefCounted {
    GDCLASS(OptimizedCalculator, RefCounted)

private:
    PackedFloat32Array results_buffer;

protected:
    static void _bind_methods() {
        // Bad: Multiple calls for multiple results
        ClassDB::bind_method(D_METHOD("calculate_single", "value"),
                           &OptimizedCalculator::calculate_single);

        // Good: Single call for batch processing
        ClassDB::bind_method(D_METHOD("calculate_batch", "values"),
                           &OptimizedCalculator::calculate_batch);
    }

public:
    // Bad: Called many times from GDScript
    float calculate_single(float value) {
        return sqrt(value * value + 1.0f);
    }

    // Good: Process entire array in one call
    PackedFloat32Array calculate_batch(const PackedFloat32Array &values) {
        results_buffer.resize(values.size());
        const float *input = values.ptr();
        float *output = results_buffer.ptrw();

        // Vectorized processing
        for (int i = 0; i < values.size(); i++) {
            output[i] = sqrt(input[i] * input[i] + 1.0f);
        }

        return results_buffer;
    }
};
```

## Memory Optimization

### Cache-Friendly Data Structures

```cpp
class ParticleSystem : public Node2D {
    GDCLASS(ParticleSystem, Node2D)

private:
    // Bad: Array of Objects (cache-unfriendly)
    // Vector<Ref<Particle>> particles;

    // Good: Structure of Arrays (cache-friendly)
    struct ParticleData {
        PackedVector2Array positions;
        PackedFloat32Array velocities_x;
        PackedFloat32Array velocities_y;
        PackedFloat32Array lifetimes;
        PackedInt32Array colors;  // Packed as integers

        int count = 0;
        int capacity = 0;

        void resize(int new_capacity) {
            capacity = new_capacity;
            positions.resize(capacity);
            velocities_x.resize(capacity);
            velocities_y.resize(capacity);
            lifetimes.resize(capacity);
            colors.resize(capacity);
        }

        void add_particle(const Vector2 &pos, const Vector2 &vel,
                         float lifetime, uint32_t color) {
            if (count >= capacity) {
                resize(capacity == 0 ? 64 : capacity * 2);
            }

            positions.set(count, pos);
            velocities_x.set(count, vel.x);
            velocities_y.set(count, vel.y);
            lifetimes.set(count, lifetime);
            colors.set(count, color);
            count++;
        }
    };

    ParticleData particles;

public:
    void _process(double delta) override {
        // SIMD-friendly processing
        Vector2 *pos = particles.positions.ptrw();
        float *vel_x = particles.velocities_x.ptrw();
        float *vel_y = particles.velocities_y.ptrw();
        float *life = particles.lifetimes.ptrw();

        int alive_count = 0;

        // Process 4 particles at once (manual vectorization)
        int simd_count = particles.count & ~3;  // Round down to multiple of 4

        for (int i = 0; i < simd_count; i += 4) {
            // Update positions
            pos[i].x += vel_x[i] * delta;
            pos[i].y += vel_y[i] * delta;
            pos[i+1].x += vel_x[i+1] * delta;
            pos[i+1].y += vel_y[i+1] * delta;
            pos[i+2].x += vel_x[i+2] * delta;
            pos[i+2].y += vel_y[i+2] * delta;
            pos[i+3].x += vel_x[i+3] * delta;
            pos[i+3].y += vel_y[i+3] * delta;

            // Update lifetimes
            life[i] -= delta;
            life[i+1] -= delta;
            life[i+2] -= delta;
            life[i+3] -= delta;
        }

        // Handle remaining particles
        for (int i = simd_count; i < particles.count; i++) {
            pos[i].x += vel_x[i] * delta;
            pos[i].y += vel_y[i] * delta;
            life[i] -= delta;
        }

        // Remove dead particles
        compact_particles();
    }

private:
    void compact_particles() {
        int write_index = 0;
        const float *life = particles.lifetimes.ptr();

        for (int read_index = 0; read_index < particles.count; read_index++) {
            if (life[read_index] > 0.0f) {
                if (write_index != read_index) {
                    // Move particle data
                    particles.positions.set(write_index, particles.positions[read_index]);
                    particles.velocities_x.set(write_index, particles.velocities_x[read_index]);
                    particles.velocities_y.set(write_index, particles.velocities_y[read_index]);
                    particles.lifetimes.set(write_index, particles.lifetimes[read_index]);
                    particles.colors.set(write_index, particles.colors[read_index]);
                }
                write_index++;
            }
        }

        particles.count = write_index;
    }
};
```

### Memory Pool Allocators

```cpp
class MemoryPool {
private:
    struct Block {
        bool in_use;
        size_t size;
        uint8_t *data;
        Block *next;
    };

    Block *free_list = nullptr;
    Vector<uint8_t> pool_memory;
    size_t pool_size;
    size_t block_size;
    int num_blocks;

public:
    MemoryPool(size_t p_block_size, int p_num_blocks)
        : block_size(p_block_size), num_blocks(p_num_blocks) {

        pool_size = block_size * num_blocks + sizeof(Block) * num_blocks;
        pool_memory.resize(pool_size);

        // Initialize free list
        uint8_t *memory = pool_memory.ptrw();
        Block *blocks = reinterpret_cast<Block*>(memory);
        uint8_t *data_start = memory + sizeof(Block) * num_blocks;

        for (int i = 0; i < num_blocks; i++) {
            blocks[i].in_use = false;
            blocks[i].size = block_size;
            blocks[i].data = data_start + i * block_size;
            blocks[i].next = (i == num_blocks - 1) ? nullptr : &blocks[i + 1];
        }

        free_list = blocks;
    }

    void* allocate(size_t size) {
        if (size > block_size || !free_list) {
            return nullptr;  // Fallback to system allocator
        }

        Block *block = free_list;
        free_list = block->next;
        block->in_use = true;
        return block->data;
    }

    void deallocate(void *ptr) {
        if (!ptr) return;

        // Find block containing this pointer
        uint8_t *memory = pool_memory.ptrw();
        Block *blocks = reinterpret_cast<Block*>(memory);

        for (int i = 0; i < num_blocks; i++) {
            if (blocks[i].data == ptr) {
                blocks[i].in_use = false;
                blocks[i].next = free_list;
                free_list = &blocks[i];
                break;
            }
        }
    }
};

class PoolAllocatedClass : public RefCounted {
    GDCLASS(PoolAllocatedClass, RefCounted)

private:
    static MemoryPool *pool;

public:
    static void initialize_pool() {
        if (!pool) {
            pool = new MemoryPool(sizeof(PoolAllocatedClass), 1000);
        }
    }

    static void cleanup_pool() {
        delete pool;
        pool = nullptr;
    }

    void* operator new(size_t size) {
        if (pool) {
            void *ptr = pool->allocate(size);
            if (ptr) return ptr;
        }
        return ::operator new(size);
    }

    void operator delete(void *ptr) {
        if (pool) {
            pool->deallocate(ptr);
            return;
        }
        ::operator delete(ptr);
    }
};

MemoryPool *PoolAllocatedClass::pool = nullptr;
```

## Algorithmic Optimizations

### Spatial Data Structures

```cpp
class QuadTree : public RefCounted {
    GDCLASS(QuadTree, RefCounted)

private:
    struct QuadNode {
        Rect2 bounds;
        Vector<int> objects;  // Object IDs
        std::unique_ptr<QuadNode> children[4];
        bool is_leaf = true;

        static const int MAX_OBJECTS = 8;
        static const int MAX_DEPTH = 6;
    };

    std::unique_ptr<QuadNode> root;
    HashMap<int, Vector2> object_positions;
    int next_object_id = 0;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("insert", "position"),
                           &QuadTree::insert);
        ClassDB::bind_method(D_METHOD("query_range", "area"),
                           &QuadTree::query_range);
        ClassDB::bind_method(D_METHOD("find_nearest", "position", "max_distance"),
                           &QuadTree::find_nearest);
    }

public:
    QuadTree() {
        root = std::make_unique<QuadNode>();
        root->bounds = Rect2(-1000, -1000, 2000, 2000);
    }

    int insert(const Vector2 &position) {
        int id = next_object_id++;
        object_positions[id] = position;
        insert_recursive(root.get(), id, position, 0);
        return id;
    }

    PackedInt32Array query_range(const Rect2 &area) {
        Vector<int> results;
        query_recursive(root.get(), area, results);

        PackedInt32Array packed_results;
        packed_results.resize(results.size());
        for (int i = 0; i < results.size(); i++) {
            packed_results.set(i, results[i]);
        }
        return packed_results;
    }

    int find_nearest(const Vector2 &position, float max_distance) {
        int nearest_id = -1;
        float nearest_distance = max_distance * max_distance;

        find_nearest_recursive(root.get(), position, nearest_id, nearest_distance);
        return nearest_id;
    }

private:
    void insert_recursive(QuadNode *node, int object_id,
                         const Vector2 &position, int depth) {
        if (node->is_leaf) {
            node->objects.append(object_id);

            if (node->objects.size() > QuadNode::MAX_OBJECTS &&
                depth < QuadNode::MAX_DEPTH) {
                subdivide(node);
                redistribute_objects(node, depth);
            }
        } else {
            int quadrant = get_quadrant(node->bounds, position);
            insert_recursive(node->children[quadrant].get(),
                           object_id, position, depth + 1);
        }
    }

    void subdivide(QuadNode *node) {
        float half_width = node->bounds.size.width * 0.5f;
        float half_height = node->bounds.size.height * 0.5f;
        float x = node->bounds.position.x;
        float y = node->bounds.position.y;

        node->children[0] = std::make_unique<QuadNode>();
        node->children[0]->bounds = Rect2(x, y, half_width, half_height);

        node->children[1] = std::make_unique<QuadNode>();
        node->children[1]->bounds = Rect2(x + half_width, y, half_width, half_height);

        node->children[2] = std::make_unique<QuadNode>();
        node->children[2]->bounds = Rect2(x, y + half_height, half_width, half_height);

        node->children[3] = std::make_unique<QuadNode>();
        node->children[3]->bounds = Rect2(x + half_width, y + half_height, half_width, half_height);

        node->is_leaf = false;
    }

    void redistribute_objects(QuadNode *node, int depth) {
        Vector<int> objects_to_redistribute = node->objects;
        node->objects.clear();

        for (int obj_id : objects_to_redistribute) {
            Vector2 pos = object_positions[obj_id];
            int quadrant = get_quadrant(node->bounds, pos);
            insert_recursive(node->children[quadrant].get(), obj_id, pos, depth + 1);
        }
    }

    int get_quadrant(const Rect2 &bounds, const Vector2 &position) {
        float mid_x = bounds.position.x + bounds.size.width * 0.5f;
        float mid_y = bounds.position.y + bounds.size.height * 0.5f;

        if (position.x < mid_x) {
            return (position.y < mid_y) ? 0 : 2;
        } else {
            return (position.y < mid_y) ? 1 : 3;
        }
    }

    void query_recursive(QuadNode *node, const Rect2 &area, Vector<int> &results) {
        if (!node->bounds.intersects(area)) {
            return;
        }

        if (node->is_leaf) {
            for (int obj_id : node->objects) {
                Vector2 pos = object_positions[obj_id];
                if (area.has_point(pos)) {
                    results.append(obj_id);
                }
            }
        } else {
            for (int i = 0; i < 4; i++) {
                query_recursive(node->children[i].get(), area, results);
            }
        }
    }

    void find_nearest_recursive(QuadNode *node, const Vector2 &position,
                               int &nearest_id, float &nearest_distance_sq) {
        float distance_to_bounds = distance_to_rect(position, node->bounds);
        if (distance_to_bounds * distance_to_bounds > nearest_distance_sq) {
            return;  // This node can't contain anything closer
        }

        if (node->is_leaf) {
            for (int obj_id : node->objects) {
                Vector2 pos = object_positions[obj_id];
                float dist_sq = position.distance_squared_to(pos);
                if (dist_sq < nearest_distance_sq) {
                    nearest_distance_sq = dist_sq;
                    nearest_id = obj_id;
                }
            }
        } else {
            // Check children in order of distance to position
            int quadrant = get_quadrant(node->bounds, position);
            find_nearest_recursive(node->children[quadrant].get(),
                                 position, nearest_id, nearest_distance_sq);

            for (int i = 0; i < 4; i++) {
                if (i != quadrant) {
                    find_nearest_recursive(node->children[i].get(),
                                         position, nearest_id, nearest_distance_sq);
                }
            }
        }
    }

    float distance_to_rect(const Vector2 &point, const Rect2 &rect) {
        float dx = MAX(0.0f, MAX(rect.position.x - point.x,
                                point.x - (rect.position.x + rect.size.width)));
        float dy = MAX(0.0f, MAX(rect.position.y - point.y,
                                point.y - (rect.position.y + rect.size.height)));
        return sqrt(dx * dx + dy * dy);
    }
};
```

## Threading and Concurrency

### Thread-Safe Data Structures

```cpp
#include <atomic>
#include <mutex>
#include <thread>

class ThreadSafeTaskQueue : public RefCounted {
    GDCLASS(ThreadSafeTaskQueue, RefCounted)

private:
    struct Task {
        std::function<void()> function;
        std::atomic<bool> completed{false};
    };

    std::vector<std::unique_ptr<Task>> tasks;
    std::mutex queue_mutex;
    std::atomic<int> next_task_index{0};
    std::atomic<int> completed_tasks{0};

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("add_task", "callable"),
                           &ThreadSafeTaskQueue::add_task);
        ClassDB::bind_method(D_METHOD("process_tasks", "max_tasks"),
                           &ThreadSafeTaskQueue::process_tasks);
        ClassDB::bind_method(D_METHOD("get_completion_ratio"),
                           &ThreadSafeTaskQueue::get_completion_ratio);
    }

public:
    void add_task(const Callable &callable) {
        auto task = std::make_unique<Task>();
        task->function = [callable]() {
            const_cast<Callable&>(callable).call();
        };

        std::lock_guard<std::mutex> lock(queue_mutex);
        tasks.push_back(std::move(task));
    }

    int process_tasks(int max_tasks) {
        int processed = 0;

        while (processed < max_tasks) {
            int task_index = next_task_index.fetch_add(1);
            if (task_index >= static_cast<int>(tasks.size())) {
                break;
            }

            Task *task = tasks[task_index].get();
            if (task && !task->completed.load()) {
                task->function();
                task->completed.store(true);
                completed_tasks.fetch_add(1);
                processed++;
            }
        }

        return processed;
    }

    float get_completion_ratio() {
        int total = static_cast<int>(tasks.size());
        if (total == 0) return 1.0f;
        return static_cast<float>(completed_tasks.load()) / total;
    }
};
```

### Lock-Free Structures

```cpp
template<typename T, size_t Size>
class LockFreeRingBuffer {
private:
    std::atomic<size_t> write_index{0};
    std::atomic<size_t> read_index{0};
    std::array<T, Size> buffer;

public:
    bool try_push(const T &item) {
        size_t current_write = write_index.load();
        size_t next_write = (current_write + 1) % Size;

        if (next_write == read_index.load()) {
            return false;  // Buffer full
        }

        buffer[current_write] = item;
        write_index.store(next_write);
        return true;
    }

    bool try_pop(T &item) {
        size_t current_read = read_index.load();

        if (current_read == write_index.load()) {
            return false;  // Buffer empty
        }

        item = buffer[current_read];
        read_index.store((current_read + 1) % Size);
        return true;
    }

    size_t size() const {
        size_t write = write_index.load();
        size_t read = read_index.load();
        return (write >= read) ? (write - read) : (Size - read + write);
    }
};

class HighPerformanceAudioProcessor : public AudioStreamPlayback {
    GDCLASS(HighPerformanceAudioProcessor, AudioStreamPlayback)

private:
    LockFreeRingBuffer<float, 8192> audio_buffer;
    std::thread processing_thread;
    std::atomic<bool> running{false};

public:
    void start_processing() {
        running.store(true);
        processing_thread = std::thread([this]() {
            while (running.load()) {
                process_audio_chunk();
                std::this_thread::sleep_for(std::chrono::microseconds(100));
            }
        });
    }

    void stop_processing() {
        running.store(false);
        if (processing_thread.joinable()) {
            processing_thread.join();
        }
    }

private:
    void process_audio_chunk() {
        // Generate audio samples
        for (int i = 0; i < 64; i++) {
            float sample = generate_sample();
            if (!audio_buffer.try_push(sample)) {
                break;  // Buffer full, skip this chunk
            }
        }
    }

    float generate_sample() {
        // High-performance audio generation
        static float phase = 0.0f;
        phase += 0.01f;
        return sin(phase);
    }
};
```

## Profiling and Benchmarking

### Built-in Performance Monitoring

```cpp
class PerformanceProfiler : public Node {
    GDCLASS(PerformanceProfiler, Node)

private:
    struct ProfileData {
        String name;
        uint64_t start_time;
        uint64_t total_time = 0;
        int call_count = 0;
        uint64_t min_time = UINT64_MAX;
        uint64_t max_time = 0;
    };

    HashMap<String, ProfileData> profiles;

protected:
    static void _bind_methods() {
        ClassDB::bind_method(D_METHOD("start_profile", "name"),
                           &PerformanceProfiler::start_profile);
        ClassDB::bind_method(D_METHOD("end_profile", "name"),
                           &PerformanceProfiler::end_profile);
        ClassDB::bind_method(D_METHOD("get_profile_report"),
                           &PerformanceProfiler::get_profile_report);
    }

public:
    void start_profile(const String &name) {
        ProfileData &data = profiles[name];
        data.name = name;
        data.start_time = OS::get_singleton()->get_ticks_usec();
    }

    void end_profile(const String &name) {
        uint64_t end_time = OS::get_singleton()->get_ticks_usec();

        if (profiles.has(name)) {
            ProfileData &data = profiles[name];
            uint64_t duration = end_time - data.start_time;

            data.total_time += duration;
            data.call_count++;
            data.min_time = MIN(data.min_time, duration);
            data.max_time = MAX(data.max_time, duration);
        }
    }

    Dictionary get_profile_report() {
        Dictionary report;

        for (const KeyValue<String, ProfileData> &E : profiles) {
            const ProfileData &data = E.value;

            Dictionary profile_info;
            profile_info["call_count"] = data.call_count;
            profile_info["total_time_us"] = static_cast<int>(data.total_time);
            profile_info["average_time_us"] = data.call_count > 0 ?
                static_cast<int>(data.total_time / data.call_count) : 0;
            profile_info["min_time_us"] = static_cast<int>(data.min_time);
            profile_info["max_time_us"] = static_cast<int>(data.max_time);

            report[data.name] = profile_info;
        }

        return report;
    }
};

// RAII profiler helper
class ScopedProfiler {
private:
    PerformanceProfiler *profiler;
    String name;

public:
    ScopedProfiler(PerformanceProfiler *p_profiler, const String &p_name)
        : profiler(p_profiler), name(p_name) {
        profiler->start_profile(name);
    }

    ~ScopedProfiler() {
        profiler->end_profile(name);
    }
};

#define PROFILE_SCOPE(profiler, name) ScopedProfiler _prof(profiler, name)

// Usage example
void expensive_operation(PerformanceProfiler *profiler) {
    PROFILE_SCOPE(profiler, "expensive_operation");

    // Do expensive work
    for (int i = 0; i < 1000000; i++) {
        // Some computation
    }
}
```

## Best Practices Summary

### 1. Minimize Cross-Boundary Calls
- Batch operations when possible
- Cache frequently accessed data
- Use PackedArrays for bulk data transfer

### 2. Optimize Memory Usage
- Use Structure of Arrays for cache efficiency
- Implement memory pools for frequent allocations
- Avoid unnecessary copying of large objects

### 3. Choose Appropriate Algorithms
- Use spatial data structures for spatial queries
- Implement proper data structures for your use case
- Consider algorithmic complexity

### 4. Threading Considerations
- Use lock-free data structures when possible
- Minimize lock contention
- Keep thread synchronization simple

### 5. Profile and Measure
- Profile real workloads, not synthetic tests
- Focus on the biggest bottlenecks first
- Measure before and after optimizations

## Conclusion

Performance optimization in GDExtension requires understanding both the engine's architecture and low-level optimization techniques. The key is to minimize cross-boundary calls, use cache-friendly data structures, and choose appropriate algorithms for your specific use case. Always profile your code to ensure optimizations are actually improving performance.
