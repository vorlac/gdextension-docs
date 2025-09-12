# Error Codes and Macros Reference

## Table of Contents
1. [Error Codes](#error-codes)
2. [Error Check Macros](#error-check-macros)
3. [Assertion Macros](#assertion-macros)
4. [Warning Macros](#warning-macros)
5. [Crash Macros](#crash-macros)
6. [Memory Debug Macros](#memory-debug-macros)
7. [Print Macros](#print-macros)
8. [Usage Examples](#usage-examples)

## Introduction

**Robust error handling for reliable extensions:** Error handling is one of the most critical aspects of GDExtension development. When your C++ code runs inside Godot, failures can crash the editor or cause unpredictable behavior in games. Proper error handling prevents crashes, provides meaningful feedback to developers, and ensures your extension behaves gracefully under unexpected conditions.

**Why error macros are essential:** Godot's error macros do much more than simple return statements. They automatically capture file names, line numbers, and function names for debugging. They integrate with the editor's error reporting system, showing meaningful messages in the console. They also include performance optimizations that mark error conditions as unlikely, helping the CPU predict the common code path.

**When to use error handling:** Use error macros at API boundaries (public methods), when validating user input, after resource allocation attempts, when calling external APIs that can fail, and whenever assumptions about data might be violated. The goal is to fail fast and provide clear information about what went wrong.

## Overview

This reference provides a complete catalog of error codes and macros available in godot-cpp. Error macros provide consistent error handling across the codebase with automatic file/line information and proper error propagation to the engine.

## Error Codes

> **Error Code Strategy**: Return `OK` (0) for success, specific error codes for expected failures. Use `FAILED` as a generic error only when no specific code applies. Error codes are compatible with GDScript and displayed in the editor's error panel.

Standard error codes from Godot:

| Category | Range | Purpose |
|----------|-------|---------|
| Success | 0 | Operation completed successfully |
| Generic | 1-5 | General failures and configuration |
| File I/O | 6-18 | File system operations |
| Resource | 19-29 | Resource and connection management |
| Data | 30-35 | Data validation and database |
| Script | 36-43 | Compilation and scripting |
| System | 44-48 | System state and special cases |

```cpp
enum Error {
    OK = 0,                          // Success
    FAILED = 1,                      // Generic failure
    ERR_UNAVAILABLE = 2,             // Resource unavailable
    ERR_UNCONFIGURED = 3,            // Not configured
    ERR_UNAUTHORIZED = 4,            // Unauthorized access
    ERR_PARAMETER_RANGE_ERROR = 5,  // Parameter out of range
    ERR_OUT_OF_MEMORY = 6,           // Out of memory
    ERR_FILE_NOT_FOUND = 7,          // File not found
    ERR_FILE_BAD_DRIVE = 8,          // Bad drive error
    ERR_FILE_BAD_PATH = 9,           // Bad path error
    ERR_FILE_NO_PERMISSION = 10,     // No permission
    ERR_FILE_ALREADY_IN_USE = 11,    // File in use
    ERR_FILE_CANT_OPEN = 12,         // Can't open file
    ERR_FILE_CANT_WRITE = 13,        // Can't write file
    ERR_FILE_CANT_READ = 14,         // Can't read file
    ERR_FILE_UNRECOGNIZED = 15,      // Unrecognized file
    ERR_FILE_CORRUPT = 16,           // Corrupt file
    ERR_FILE_MISSING_DEPENDENCIES = 17, // Missing dependencies
    ERR_FILE_EOF = 18,               // End of file
    ERR_CANT_OPEN = 19,              // Can't open
    ERR_CANT_CREATE = 20,            // Can't create
    ERR_QUERY_FAILED = 21,           // Query failed
    ERR_ALREADY_IN_USE = 22,         // Already in use
    ERR_LOCKED = 23,                 // Locked
    ERR_TIMEOUT = 24,                // Timeout
    ERR_CANT_CONNECT = 25,           // Can't connect
    ERR_CANT_RESOLVE = 26,           // Can't resolve
    ERR_CONNECTION_ERROR = 27,       // Connection error
    ERR_CANT_ACQUIRE_RESOURCE = 28,  // Can't acquire resource
    ERR_CANT_FORK = 29,              // Can't fork process
    ERR_INVALID_DATA = 30,           // Invalid data
    ERR_INVALID_PARAMETER = 31,      // Invalid parameter
    ERR_ALREADY_EXISTS = 32,         // Already exists
    ERR_DOES_NOT_EXIST = 33,         // Does not exist
    ERR_DATABASE_CANT_READ = 34,     // Database read error
    ERR_DATABASE_CANT_WRITE = 35,    // Database write error
    ERR_COMPILATION_FAILED = 36,     // Compilation failed
    ERR_METHOD_NOT_FOUND = 37,       // Method not found
    ERR_LINK_FAILED = 38,            // Link failed
    ERR_SCRIPT_FAILED = 39,          // Script failed
    ERR_CYCLIC_LINK = 40,            // Cyclic link
    ERR_INVALID_DECLARATION = 41,    // Invalid declaration
    ERR_DUPLICATE_SYMBOL = 42,       // Duplicate symbol
    ERR_PARSE_ERROR = 43,            // Parse error
    ERR_BUSY = 44,                   // Busy
    ERR_SKIP = 45,                   // Skip
    ERR_HELP = 46,                   // Help
    ERR_BUG = 47,                    // Bug
    ERR_PRINTER_ON_FIRE = 48,        // Printer on fire (humorous)
};
```

## Error Check Macros

> **Macro Selection Guide**: Use `_V` suffix when you need to return a value, `_MSG` suffix for custom error context, `_EDMSG` for editor-visible errors. Combine suffixes as needed: `ERR_FAIL_COND_V_MSG` returns a value with a custom message.

### Basic Error Checking

```cpp
// Check condition and return if false
ERR_FAIL_COND(condition)
// Usage: ERR_FAIL_COND(ptr == nullptr);

// Check condition and return value if false
ERR_FAIL_COND_V(condition, return_value)
// Usage: ERR_FAIL_COND_V(index < 0, ERR_INVALID_PARAMETER);

// Check condition with custom message
ERR_FAIL_COND_MSG(condition, message)
// Usage: ERR_FAIL_COND_MSG(size > MAX_SIZE, "Size exceeds maximum");

// Check condition, return value with message
ERR_FAIL_COND_V_MSG(condition, return_value, message)
// Usage: ERR_FAIL_COND_V_MSG(fd < 0, FAILED, "Invalid file descriptor");
```

### Null Checking

> **When to Use**: Apply null checks at public API boundaries, after dynamic casts, and before dereferencing pointers from external sources. Skip null checks for internal functions where null is impossible by design.

```cpp
// Check for null and return
ERR_FAIL_NULL(pointer)
// Usage: ERR_FAIL_NULL(p_node);

// Check for null and return value
ERR_FAIL_NULL_V(pointer, return_value)
// Usage: ERR_FAIL_NULL_V(p_data, nullptr);

// Check for null with message
ERR_FAIL_NULL_MSG(pointer, message)
// Usage: ERR_FAIL_NULL_MSG(p_resource, "Resource is null");

// Check for null, return value with message
ERR_FAIL_NULL_V_MSG(pointer, return_value, message)
// Usage: ERR_FAIL_NULL_V_MSG(p_buffer, FAILED, "Buffer allocation failed");
```

### Index Bounds Checking

> **Performance Note**: Index macros compile to simple comparisons in release builds (2-3 cycles). Always use these instead of manual checks for consistency and automatic error reporting.

```cpp
// Check index bounds and return
ERR_FAIL_INDEX(index, size)
// Usage: ERR_FAIL_INDEX(idx, array.size());

// Check index bounds and return value
ERR_FAIL_INDEX_V(index, size, return_value)
// Usage: ERR_FAIL_INDEX_V(pos, length, nullptr);

// Check index bounds with message
ERR_FAIL_INDEX_MSG(index, size, message)
// Usage: ERR_FAIL_INDEX_MSG(id, count, "ID out of range");

// Check index bounds, return value with message
ERR_FAIL_INDEX_V_MSG(index, size, return_value, message)
// Usage: ERR_FAIL_INDEX_V_MSG(slot, max_slots, FAILED, "Invalid slot");
```

### Unsigned Index Checking

```cpp
// Check unsigned index bounds
ERR_FAIL_UNSIGNED_INDEX(index, size)
// Usage: ERR_FAIL_UNSIGNED_INDEX(offset, buffer_size);

// Check unsigned index and return value
ERR_FAIL_UNSIGNED_INDEX_V(index, size, return_value)
// Usage: ERR_FAIL_UNSIGNED_INDEX_V(pos, len, 0);

// Check unsigned index with message
ERR_FAIL_UNSIGNED_INDEX_MSG(index, size, message)
// Usage: ERR_FAIL_UNSIGNED_INDEX_MSG(idx, count, "Index overflow");

// Check unsigned index, return value with message
ERR_FAIL_UNSIGNED_INDEX_V_MSG(index, size, return_value, message)
// Usage: ERR_FAIL_UNSIGNED_INDEX_V_MSG(i, n, nullptr, "Out of bounds");
```

### Direct Failure

```cpp
// Fail immediately
ERR_FAIL()
// Usage: ERR_FAIL(); // Unreachable code

// Fail and return value
ERR_FAIL_V(return_value)
// Usage: ERR_FAIL_V(ERR_BUG);

// Fail with message
ERR_FAIL_MSG(message)
// Usage: ERR_FAIL_MSG("This should never happen");

// Fail with value and message
ERR_FAIL_V_MSG(return_value, message)
// Usage: ERR_FAIL_V_MSG(FAILED, "Critical error occurred");
```

### Conditional Continuation

```cpp
// Continue if condition is true
ERR_CONTINUE(condition)
// Usage: ERR_CONTINUE(item == nullptr);

// Continue with message
ERR_CONTINUE_MSG(condition, message)
// Usage: ERR_CONTINUE_MSG(!valid, "Skipping invalid item");

// Break if condition is true
ERR_BREAK(condition)
// Usage: ERR_BREAK(done);

// Break with message
ERR_BREAK_MSG(condition, message)
// Usage: ERR_BREAK_MSG(limit_reached, "Limit exceeded");
```

## Assertion Macros

### Development Assertions

```cpp
// Debug assertion (stripped in release)
DEV_ASSERT(condition)
// Usage: DEV_ASSERT(ptr != nullptr);

// Debug assertion with message
DEBUG_ASSERT(condition, message)
// Usage: DEBUG_ASSERT(size > 0, "Size must be positive");

// Development-only check
#ifdef DEV_ENABLED
    DEV_CHECK(expensive_validation());
#endif
```

### Release Assertions

```cpp
// Assertion in all builds
CRASH_COND(condition)
// Usage: CRASH_COND(critical_ptr == nullptr);

// Assertion with message
CRASH_COND_MSG(condition, message)
// Usage: CRASH_COND_MSG(state == INVALID, "Invalid state");
```

## Warning Macros

### Warning Output

```cpp
// Print warning
WARN_PRINT(message)
// Usage: WARN_PRINT("Deprecated function called");

// Print warning once
WARN_PRINT_ONCE(message)
// Usage: WARN_PRINT_ONCE("This warning appears only once");

// Conditional warning
WARN_PRINT_COND(condition, message)
// Usage: WARN_PRINT_COND(value < 0, "Negative value detected");

// Warning with formatted message
WARN_PRINT_FMT(format, ...)
// Usage: WARN_PRINT_FMT("Value %d is out of range [%d, %d]", val, min, max);
```

### Deprecation Warnings

```cpp
// Mark as deprecated
WARN_DEPRECATED
// Usage: WARN_DEPRECATED;

// Deprecated with message
WARN_DEPRECATED_MSG(message)
// Usage: WARN_DEPRECATED_MSG("Use new_function() instead");
```

## Crash Macros

### Immediate Crash

```cpp
// Crash immediately
CRASH_NOW()
// Usage: CRASH_NOW(); // Unrecoverable error

// Crash with message
CRASH_NOW_MSG(message)
// Usage: CRASH_NOW_MSG("Fatal error: corrupted data");

// Platform-specific crash
#ifdef DEBUG_ENABLED
    GENERATE_TRAP()  // Debugger breakpoint
#endif
```

## Memory Debug Macros

### Memory Tracking

```cpp
// Track memory allocation
#ifdef DEBUG_ENABLED
    MEM_ALLOC(size)
    // Usage: MEM_ALLOC(sizeof(MyClass));

    MEM_FREE(size)
    // Usage: MEM_FREE(sizeof(MyClass));

    MEM_REALLOC(old_size, new_size)
    // Usage: MEM_REALLOC(old_capacity, new_capacity);
#endif
```

### Safe Memory Operations

```cpp
// Safe delete with null check
SAFE_DELETE(pointer)
// Usage: SAFE_DELETE(p_object);

// Safe array delete
SAFE_DELETE_ARRAY(pointer)
// Usage: SAFE_DELETE_ARRAY(p_buffer);

// Safe reference release
SAFE_RELEASE(reference)
// Usage: SAFE_RELEASE(ref_counted_obj);
```

## Print Macros

### Standard Output

```cpp
// Print message
print_line(message)
// Usage: print_line("Processing started");

// Print formatted
print_line_fmt(format, ...)
// Usage: print_line_fmt("Progress: %d%%", percentage);

// Print error
print_error(message)
// Usage: print_error("Operation failed");

// Print verbose (debug only)
print_verbose(message)
// Usage: print_verbose("Debug: entering function");
```

### Conditional Printing

```cpp
// Print if verbose mode
VERBOSE_PRINT(message)
// Usage: VERBOSE_PRINT("Detailed debug info");

// Print in editor only
#ifdef TOOLS_ENABLED
    EDITOR_PRINT(message)
#endif
```

## Usage Examples

### Error Handling Pattern

```cpp
Error MyClass::load_resource(const String &p_path) {
    // Validate input
    ERR_FAIL_COND_V_MSG(p_path.is_empty(), ERR_INVALID_PARAMETER,
                        "Path cannot be empty");

    // Open file
    Ref<FileAccess> file = FileAccess::open(p_path, FileAccess::READ);
    ERR_FAIL_NULL_V_MSG(file, ERR_FILE_CANT_OPEN,
                        vformat("Cannot open file: %s", p_path));

    // Read header
    uint32_t magic = file->get_32();
    ERR_FAIL_COND_V_MSG(magic != EXPECTED_MAGIC, ERR_FILE_CORRUPT,
                        "Invalid file format");

    // Allocate buffer
    uint64_t size = file->get_length();
    ERR_FAIL_COND_V_MSG(size > MAX_FILE_SIZE, ERR_OUT_OF_MEMORY,
                        "File too large");

    uint8_t *buffer = (uint8_t *)memalloc(size);
    ERR_FAIL_NULL_V_MSG(buffer, ERR_OUT_OF_MEMORY,
                        "Failed to allocate memory");

    // Read data
    uint64_t read = file->get_buffer(buffer, size);
    if (read != size) {
        memfree(buffer);
        ERR_FAIL_V_MSG(ERR_FILE_CANT_READ, "Incomplete read");
    }

    // Process data
    Error err = process_data(buffer, size);
    memfree(buffer);

    ERR_FAIL_COND_V_MSG(err != OK, err,
                        "Failed to process data");

    return OK;
}
```

### Validation Pattern

```cpp
void MyClass::set_property(int p_value) {
    // Range validation
    ERR_FAIL_COND_MSG(p_value < MIN_VALUE || p_value > MAX_VALUE,
                      vformat("Value must be between %d and %d",
                              MIN_VALUE, MAX_VALUE));

    // State validation
    ERR_FAIL_COND_MSG(state != STATE_READY,
                      "Cannot set property in current state");

    // Apply value
    property = p_value;

    // Warn about edge cases
    if (p_value == MIN_VALUE) {
        WARN_PRINT("Using minimum value may cause performance issues");
    }
}
```

### Debug Pattern

```cpp
void MyClass::complex_algorithm() {
    DEV_ASSERT(initialized);  // Debug-only check

    #ifdef DEV_ENABLED
    Timer timer;
    timer.start();
    #endif

    for (int i = 0; i < count; i++) {
        ERR_CONTINUE_MSG(!items[i].is_valid(),
                         vformat("Invalid item at index %d", i));

        process_item(items[i]);

        #ifdef DEV_ENABLED
        if (timer.get_elapsed() > 1.0) {
            WARN_PRINT_ONCE("Algorithm taking too long");
        }
        #endif
    }

    #ifdef DEV_ENABLED
    print_verbose(vformat("Algorithm completed in %.3f ms",
                         timer.get_elapsed() * 1000));
    #endif
}
```

## Macro Expansion Reference

### Example Expansion

```cpp
// ERR_FAIL_NULL(ptr) expands to approximately:
do {
    if (unlikely(ptr == nullptr)) {
        _err_print_error(__FUNCTION__, __FILE__, __LINE__,
                        "Condition \"ptr == nullptr\" is true.");
        return;
    }
} while (0)

// ERR_FAIL_COND_V_MSG(cond, ret, msg) expands to approximately:
do {
    if (unlikely(cond)) {
        _err_print_error(__FUNCTION__, __FILE__, __LINE__, msg);
        return ret;
    }
} while (0)
```

## Summary

The error macro system provides:
- **80+ macros** for comprehensive error handling
- **Conditional compilation** for debug/release differences
- **Source location tracking** for debugging
- **Performance optimization** with unlikely() hints
- **Type safety** through template implementations
- **Consistent patterns** across the codebase

These macros ensure robust error handling while maintaining performance in release builds.
