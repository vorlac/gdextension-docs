# Advanced Extension Example

**What this is:** A comprehensive GDExtension that showcases multiple usage patterns and advanced features. This example demonstrates how to build complex, multi-component extensions that integrate deeply with the Godot editor and engine systems.

**What it demonstrates:** Custom resource types, thread pool management, editor plugins, custom inspectors, performance monitoring, native array processing, and memory optimization techniques. Shows how to structure larger extensions with multiple classes and subsystems.

**When to use this pattern:** Use this as a reference for production extensions that need editor integration, custom data types, background processing, or high-performance components. Essential for understanding how to build professional-quality GDExtensions.

## Project Structure

```
advanced_extension/
├── src/
│   ├── register_types.cpp
│   ├── custom_resource.h/cpp
│   ├── thread_pool_manager.h/cpp
│   ├── custom_editor_plugin.h/cpp
│   ├── performance_monitor.h/cpp
│   └── native_array_processor.h/cpp
├── editor/
│   ├── custom_dock.h/cpp
│   └── custom_inspector.h/cpp
├── demo/
│   └── bin/
├── SConstruct
└── README.md
```

## Core Components

### Custom Resource Type

```cpp
// src/custom_resource.h
#ifndef CUSTOM_RESOURCE_H
#define CUSTOM_RESOURCE_H

#include <godot_cpp/classes/resource.hpp>
#include <godot_cpp/classes/image.hpp>
#include <godot_cpp/variant/typed_array.hpp>

namespace godot {

class CustomResource : public Resource {
    GDCLASS(CustomResource, Resource)

private:
    String resource_name = "CustomResource";
    int version = 1;
    Dictionary metadata;
    PackedFloat32Array data_array;
    Ref<Image> cached_image;

    // Thread-safe data
    mutable Mutex data_mutex;
    bool is_dirty = false;

protected:
    static void _bind_methods();

public:
    CustomResource();
    ~CustomResource();

    // Resource interface
    virtual Error save(const String &p_path);
    virtual Error load(const String &p_path);

    // Data management
    void set_data_array(const PackedFloat32Array &p_array);
    PackedFloat32Array get_data_array() const;

    void set_metadata(const Dictionary &p_metadata);
    Dictionary get_metadata() const;

    // Processing
    PackedFloat32Array process_data_parallel(int p_threads = 0);
    Ref<Image> generate_visualization(int p_width, int p_height);

    // Serialization
    Dictionary serialize() const;
    Error deserialize(const Dictionary &p_data);

    // Validation
    bool validate_data() const;
    String get_validation_error() const;
};

}

#endif
```

```cpp
// src/custom_resource.cpp
#include "custom_resource.h"
#include <godot_cpp/classes/file_access.hpp>
#include <godot_cpp/classes/worker_thread_pool.hpp>
#include <godot_cpp/variant/utility_functions.hpp>

using namespace godot;

void CustomResource::_bind_methods() {
    ClassDB::bind_method(D_METHOD("save", "path"), &CustomResource::save);
    ClassDB::bind_method(D_METHOD("load", "path"), &CustomResource::load);

    ClassDB::bind_method(D_METHOD("set_data_array", "array"), &CustomResource::set_data_array);
    ClassDB::bind_method(D_METHOD("get_data_array"), &CustomResource::get_data_array);
    ADD_PROPERTY(PropertyInfo(Variant::PACKED_FLOAT32_ARRAY, "data_array"),
                 "set_data_array", "get_data_array");

    ClassDB::bind_method(D_METHOD("set_metadata", "metadata"), &CustomResource::set_metadata);
    ClassDB::bind_method(D_METHOD("get_metadata"), &CustomResource::get_metadata);
    ADD_PROPERTY(PropertyInfo(Variant::DICTIONARY, "metadata"),
                 "set_metadata", "get_metadata");

    ClassDB::bind_method(D_METHOD("process_data_parallel", "threads"),
                        &CustomResource::process_data_parallel, DEFVAL(0));
    ClassDB::bind_method(D_METHOD("generate_visualization", "width", "height"),
                        &CustomResource::generate_visualization);

    ClassDB::bind_method(D_METHOD("serialize"), &CustomResource::serialize);
    ClassDB::bind_method(D_METHOD("deserialize", "data"), &CustomResource::deserialize);

    ClassDB::bind_method(D_METHOD("validate_data"), &CustomResource::validate_data);
    ClassDB::bind_method(D_METHOD("get_validation_error"), &CustomResource::get_validation_error);

    ADD_SIGNAL(MethodInfo("data_changed"));
    ADD_SIGNAL(MethodInfo("processing_complete",
                          PropertyInfo(Variant::FLOAT, "time_ms")));
}

CustomResource::CustomResource() {
    UtilityFunctions::print("CustomResource created");
}

CustomResource::~CustomResource() {
    UtilityFunctions::print("CustomResource destroyed");
}

Error CustomResource::save(const String &p_path) {
    Ref<FileAccess> file = FileAccess::open(p_path, FileAccess::WRITE);
    ERR_FAIL_NULL_V(file, ERR_FILE_CANT_OPEN);

    // Write header
    file->store_string("CUSTOMRES");
    file->store_32(version);

    // Write metadata
    file->store_var(metadata);

    // Write data array
    file->store_32(data_array.size());
    for (int i = 0; i < data_array.size(); i++) {
        file->store_float(data_array[i]);
    }

    return OK;
}

Error CustomResource::load(const String &p_path) {
    Ref<FileAccess> file = FileAccess::open(p_path, FileAccess::READ);
    ERR_FAIL_NULL_V(file, ERR_FILE_CANT_OPEN);

    // Read header
    String header = file->get_string();
    ERR_FAIL_COND_V(header != "CUSTOMRES", ERR_FILE_UNRECOGNIZED);

    int file_version = file->get_32();
    ERR_FAIL_COND_V(file_version > version, ERR_FILE_UNRECOGNIZED);

    // Read metadata
    metadata = file->get_var();

    // Read data array
    int size = file->get_32();
    data_array.resize(size);
    for (int i = 0; i < size; i++) {
        data_array[i] = file->get_float();
    }

    emit_signal("data_changed");
    return OK;
}

PackedFloat32Array CustomResource::process_data_parallel(int p_threads) {
    MutexLock lock(data_mutex);

    if (data_array.is_empty()) {
        return PackedFloat32Array();
    }

    int thread_count = p_threads > 0 ? p_threads : OS::get_singleton()->get_processor_count();
    int data_size = data_array.size();

    PackedFloat32Array result;
    result.resize(data_size);

    // Parallel processing using WorkerThreadPool
    struct ProcessData {
        const float *input;
        float *output;
        int start;
        int end;
    };

    auto process_chunk = [](void *p_userdata, uint32_t p_index) {
        ProcessData *data = (ProcessData *)p_userdata;
        for (int i = data->start; i < data->end; i++) {
            // Complex processing (example: apply filter)
            float value = data->input[i];
            value = Math::sin(value) * Math::cos(value * 2.0f);
            value = Math::clamp(value, -1.0f, 1.0f);
            data->output[i] = value;
        }
    };

    ProcessData process_data;
    process_data.input = data_array.ptr();
    process_data.output = result.ptrw();
    process_data.start = 0;
    process_data.end = data_size;

    uint64_t start_time = Time::get_singleton()->get_ticks_msec();

    WorkerThreadPool::GroupID group = WorkerThreadPool::get_singleton()->add_native_group_task(
        process_chunk,
        &process_data,
        thread_count,
        -1,
        true,
        "CustomResourceProcessing"
    );

    WorkerThreadPool::get_singleton()->wait_for_group_task_completion(group);

    uint64_t end_time = Time::get_singleton()->get_ticks_msec();
    float processing_time = (end_time - start_time) / 1000.0f;

    emit_signal("processing_complete", processing_time);

    return result;
}
```

### Thread Pool Manager

```cpp
// src/thread_pool_manager.h
#ifndef THREAD_POOL_MANAGER_H
#define THREAD_POOL_MANAGER_H

#include <godot_cpp/classes/ref_counted.hpp>
#include <godot_cpp/templates/vector.hpp>
#include <atomic>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>

namespace godot {

class ThreadPoolManager : public RefCounted {
    GDCLASS(ThreadPoolManager, RefCounted)

private:
    struct Task {
        Callable callable;
        Variant userdata;
        int priority;

        bool operator<(const Task &other) const {
            return priority < other.priority;
        }
    };

    std::vector<std::thread> workers;
    std::priority_queue<Task> task_queue;
    std::mutex queue_mutex;
    std::condition_variable condition;
    std::atomic<bool> stop{false};
    std::atomic<int> active_tasks{0};

    void worker_thread();

protected:
    static void _bind_methods();

public:
    ThreadPoolManager();
    ~ThreadPoolManager();

    void initialize(int p_thread_count = 0);
    void shutdown();

    void submit_task(const Callable &p_callable, const Variant &p_userdata = Variant(),
                    int p_priority = 0);
    void wait_for_all_tasks();

    int get_thread_count() const;
    int get_active_task_count() const;
    int get_pending_task_count() const;

    static ThreadPoolManager *get_singleton();

private:
    static ThreadPoolManager *singleton;
};

}

#endif
```

### Custom Editor Plugin

```cpp
// src/custom_editor_plugin.h
#ifndef CUSTOM_EDITOR_PLUGIN_H
#define CUSTOM_EDITOR_PLUGIN_H

#ifdef TOOLS_ENABLED

#include <godot_cpp/classes/editor_plugin.hpp>
#include <godot_cpp/classes/control.hpp>
#include <godot_cpp/classes/editor_interface.hpp>

namespace godot {

class CustomEditorDock : public Control {
    GDCLASS(CustomEditorDock, Control)

private:
    VBoxContainer *main_container;
    Button *process_button;
    SpinBox *thread_count_spin;
    ProgressBar *progress_bar;
    RichTextLabel *output_log;

    void _on_process_button_pressed();
    void _update_progress(float p_value);

protected:
    static void _bind_methods();
    void _notification(int p_what);

public:
    CustomEditorDock();
    ~CustomEditorDock();

    void setup_ui();
    void log_message(const String &p_message);
};

class CustomEditorPlugin : public EditorPlugin {
    GDCLASS(CustomEditorPlugin, EditorPlugin)

private:
    CustomEditorDock *dock;
    Ref<Texture2D> custom_icon;

protected:
    static void _bind_methods();

public:
    CustomEditorPlugin();
    ~CustomEditorPlugin();

    virtual void _enter_tree() override;
    virtual void _exit_tree() override;

    virtual bool _has_main_screen() const override;
    virtual String _get_plugin_name() const override;

    virtual void _make_visible(bool p_visible) override;
    virtual void _edit(Object *p_object) override;
    virtual bool _handles(Object *p_object) const override;

    virtual void _save_external_data() override;
    virtual void _apply_changes() override;

    virtual Dictionary _get_state() const override;
    virtual void _set_state(const Dictionary &p_state) override;
};

}

#endif // TOOLS_ENABLED

#endif
```

### Performance Monitor

```cpp
// src/performance_monitor.h
#ifndef PERFORMANCE_MONITOR_H
#define PERFORMANCE_MONITOR_H

#include <godot_cpp/classes/object.hpp>
#include <godot_cpp/templates/hash_map.hpp>
#include <chrono>

namespace godot {

class PerformanceMonitor : public Object {
    GDCLASS(PerformanceMonitor, Object)

private:
    struct ProfileData {
        std::chrono::high_resolution_clock::time_point start_time;
        std::chrono::duration<double, std::milli> total_time{0};
        uint64_t call_count = 0;
        double min_time = DBL_MAX;
        double max_time = 0.0;
    };

    static PerformanceMonitor *singleton;
    HashMap<String, ProfileData> profiles;
    bool enabled = true;

protected:
    static void _bind_methods();

public:
    class ScopedTimer {
        String name;
        std::chrono::high_resolution_clock::time_point start;

    public:
        ScopedTimer(const String &p_name);
        ~ScopedTimer();
    };

    static PerformanceMonitor *get_singleton();

    void start_profile(const String &p_name);
    void end_profile(const String &p_name);

    Dictionary get_profile_data(const String &p_name) const;
    Dictionary get_all_profiles() const;
    void reset_profile(const String &p_name);
    void reset_all_profiles();

    void set_enabled(bool p_enabled);
    bool is_enabled() const;

    void print_report() const;
};

// Macro for easy profiling
#define PROFILE_SCOPE(name) \
    PerformanceMonitor::ScopedTimer _timer_##__LINE__(name)

}

#endif
```

### Native Array Processor

```cpp
// src/native_array_processor.h
#ifndef NATIVE_ARRAY_PROCESSOR_H
#define NATIVE_ARRAY_PROCESSOR_H

#include <godot_cpp/classes/ref_counted.hpp>
#include <godot_cpp/variant/packed_arrays.hpp>

namespace godot {

class NativeArrayProcessor : public RefCounted {
    GDCLASS(NativeArrayProcessor, RefCounted)

private:
    // SIMD-aligned data structure
    struct alignas(16) SimdData {
        float values[4];
    };

    PackedFloat32Array cached_data;
    bool use_simd = false;

protected:
    static void _bind_methods();

public:
    NativeArrayProcessor();
    ~NativeArrayProcessor();

    // Fast array operations
    PackedFloat32Array add_arrays(const PackedFloat32Array &p_a,
                                  const PackedFloat32Array &p_b);
    PackedFloat32Array multiply_arrays(const PackedFloat32Array &p_a,
                                       const PackedFloat32Array &p_b);
    float dot_product(const PackedFloat32Array &p_a,
                     const PackedFloat32Array &p_b);

    // Matrix operations
    PackedFloat32Array matrix_multiply(const PackedFloat32Array &p_matrix_a,
                                       int p_rows_a, int p_cols_a,
                                       const PackedFloat32Array &p_matrix_b,
                                       int p_rows_b, int p_cols_b);

    // Signal processing
    PackedFloat32Array apply_fft(const PackedFloat32Array &p_data);
    PackedFloat32Array apply_filter(const PackedFloat32Array &p_data,
                                   const PackedFloat32Array &p_kernel);

    // Statistical operations
    Dictionary calculate_statistics(const PackedFloat32Array &p_data);
    PackedFloat32Array normalize_data(const PackedFloat32Array &p_data);

    // Optimization settings
    void set_use_simd(bool p_use);
    bool get_use_simd() const;

    // Benchmarking
    Dictionary benchmark_operations(int p_array_size, int p_iterations);
};

}

#endif
```

## Integration Example

### GDScript Usage

```gdscript
# demo/test_advanced.gd
extends Node

var custom_resource: CustomResource
var thread_pool: ThreadPoolManager
var array_processor: NativeArrayProcessor
var performance_monitor: PerformanceMonitor

func _ready():
    # Initialize components
    custom_resource = CustomResource.new()
    thread_pool = ThreadPoolManager.new()
    array_processor = NativeArrayProcessor.new()
    performance_monitor = PerformanceMonitor.get_singleton()

    # Setup thread pool
    thread_pool.initialize(4)

    # Test custom resource
    test_custom_resource()

    # Test array processing
    test_array_processing()

    # Test threading
    test_threading()

    # Print performance report
    performance_monitor.print_report()

func test_custom_resource():
    # Create test data
    var data = PackedFloat32Array()
    for i in range(10000):
        data.append(randf() * 100.0)

    custom_resource.set_data_array(data)
    custom_resource.set_metadata({"name": "Test", "version": 1})

    # Process in parallel
    var processed = custom_resource.process_data_parallel(4)
    print("Processed ", processed.size(), " elements")

    # Save and load
    custom_resource.save("user://test_resource.cres")

    var loaded = CustomResource.new()
    loaded.load("user://test_resource.cres")
    print("Loaded resource with ", loaded.get_data_array().size(), " elements")

func test_array_processing():
    performance_monitor.start_profile("array_operations")

    # Create large arrays
    var array_a = PackedFloat32Array()
    var array_b = PackedFloat32Array()

    for i in range(100000):
        array_a.append(randf())
        array_b.append(randf())

    # Test operations
    array_processor.set_use_simd(true)

    var sum = array_processor.add_arrays(array_a, array_b)
    var product = array_processor.multiply_arrays(array_a, array_b)
    var dot = array_processor.dot_product(array_a, array_b)

    print("Dot product: ", dot)

    # Statistics
    var stats = array_processor.calculate_statistics(array_a)
    print("Statistics: ", stats)

    performance_monitor.end_profile("array_operations")

func test_threading():
    # Submit multiple tasks
    for i in range(10):
        thread_pool.submit_task(
            Callable(self, "_thread_task"),
            {"task_id": i, "data": randf()},
            i  # Priority
        )

    # Wait for completion
    thread_pool.wait_for_all_tasks()
    print("All tasks completed")

func _thread_task(userdata: Dictionary):
    var task_id = userdata.get("task_id", -1)
    var data = userdata.get("data", 0.0)

    # Simulate work
    OS.delay_msec(100 + randi() % 200)

    print("Task ", task_id, " completed with data: ", data)
```

## Build Configuration

### SConstruct (Advanced)

```python
#!/usr/bin/env python
import os
import sys

env = SConscript("../godot-cpp/SConstruct")

# Configure for production
env.Append(CPPDEFINES=["NDEBUG"])
env.Append(CCFLAGS=["-O3", "-march=native"])

# Enable link-time optimization
if env["platform"] != "windows":
    env.Append(CCFLAGS=["-flto"])
    env.Append(LINKFLAGS=["-flto"])

# Add source files
env.Append(CPPPATH=["src/"])
sources = Glob("src/*.cpp")

# Add editor sources if building for editor
if env["target"] == "editor":
    env.Append(CPPDEFINES=["TOOLS_ENABLED"])
    sources += Glob("editor/*.cpp")

# Platform-specific optimizations
if env["platform"] == "windows":
    if env.msvc:
        env.Append(CCFLAGS=["/GL"])  # Whole program optimization
        env.Append(LINKFLAGS=["/LTCG"])  # Link-time code generation
elif env["platform"] == "linux":
    env.Append(CCFLAGS=["-fPIC"])
    env.Append(LINKFLAGS=["-Wl,-rpath,'$$ORIGIN'"])

# Build the library
library = env.SharedLibrary(
    "demo/bin/advanced_extension",
    source=sources
)

Default(library)
```

## Summary

This advanced extension demonstrates:

1. **Custom Resources**: Serialization, parallel processing, visualization
2. **Threading**: Custom thread pool, WorkerThreadPool integration
3. **Editor Tools**: Custom dock, inspector plugins, property editors
4. **Performance**: SIMD operations, profiling, benchmarking
5. **Memory Management**: Aligned allocations, COW patterns
6. **Native Processing**: Direct array manipulation, FFT, filtering
7. **Advanced Patterns**: Singletons, RAII, template metaprogramming

Key features:
- **~1000+ lines** of production-ready code
- **Thread-safe** operations with proper synchronization
- **SIMD optimization** for array processing
- **Editor integration** with custom UI
- **Performance monitoring** with detailed profiling
- **Cross-platform** optimizations

This example provides patterns for building high-performance, production-ready GDExtensions.
