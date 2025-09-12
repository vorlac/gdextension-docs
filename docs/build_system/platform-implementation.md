# Platform-Specific Implementation Internal Documentation

## Overview

godot-cpp implements platform-specific build configurations through modular Python scripts in the `tools/` directory. Each platform has unique requirements for compilers, linkers, SDKs, and runtime libraries.

### Platform Architecture
```
Platform Detection (godotcpp.py)
    ↓
Platform Tool Module (platform.py)
    ↓
Common Compiler Flags (common_compiler_flags.py)
    ↓
Platform-Specific Configuration
    ├── Compiler Selection
    ├── Architecture Flags
    ├── SDK/Toolchain Setup
    └── Runtime Library Config
```

## Windows Platform

### Compiler Selection (MSVC vs MinGW)

Windows supports two distinct toolchains:

```python
# tools/windows.py:90-106
def generate(env):
    if not env["use_mingw"] and msvc.exists(env):
        # MSVC Configuration
        env["is_msvc"] = True
        msvc.generate(env)
        env.Tool("msvc")
        env.Tool("mslib")
        env.Tool("mslink")

        # MSVC-specific defines
        env.Append(CPPDEFINES=[
            "TYPED_METHOD_BIND",  # Required for method binding
            "NOMINMAX"            # Prevent Windows.h min/max macros
        ])
        env.Append(CCFLAGS=["/utf-8"])  # UTF-8 source files
        env.Append(LINKFLAGS=["/WX"])   # Linker warnings as errors
```

### MSVC Runtime Library Configuration

```python
# tools/windows.py:121-128
if env["debug_crt"]:
    # Debug CRT with DLL runtime (thread_local requires DLL runtime)
    env.AppendUnique(CCFLAGS=["/MDd"])
else:
    if env["use_static_cpp"]:
        env.AppendUnique(CCFLAGS=["/MT"])  # Static runtime
    else:
        env.AppendUnique(CCFLAGS=["/MD"])  # DLL runtime
```

### MinGW Cross-Compilation

```python
# tools/windows.py:162-189
# Architecture-specific prefixes
if env["arch"] == "x86_64":
    prefix += "x86_64"
elif env["arch"] == "arm64":
    prefix += "aarch64"
elif env["arch"] == "arm32":
    prefix += "armv7"
elif env["arch"] == "x86_32":
    prefix += "i686"

# Compiler selection (GCC or LLVM)
if env["use_llvm"]:
    env["CXX"] = prefix + "-w64-mingw32-clang++"
    env["CC"] = prefix + "-w64-mingw32-clang"
    env["AR"] = prefix + "-w64-mingw32-llvm-ar"
else:
    env["CXX"] = prefix + "-w64-mingw32-g++"
    env["CC"] = prefix + "-w64-mingw32-gcc"
    env["AR"] = prefix + "-w64-mingw32-gcc-ar"

# Static linking for portability
if env["use_static_cpp"]:
    env.Append(LINKFLAGS=[
        "-static",
        "-static-libgcc",
        "-static-libstdc++"
    ])
```

### MSVC Output Silencing

```python
# tools/windows.py:10-73
def silence_msvc(env):
    """Sophisticated output filtering for MSVC"""
    # Captures cl.exe and link.exe output
    # Filters compilation noise while preserving errors/warnings

    def spawn_capture(sh, escape, cmd, args, env):
        if args[0] not in ["cl", "link"]:
            return old_spawn(sh, escape, cmd, args, env)

        # Redirect to temp file
        tmp_stdout, tmp_stdout_name = tempfile.mkstemp()
        args.append(f">{tmp_stdout_name}")
        ret = old_spawn(sh, escape, cmd, args, env)

        # Process output, filter noise, keep errors
        # Regex patterns identify actual errors vs normal output
```

## Linux Platform

### Compiler Configuration

```python
# tools/linux.py:15-22
def generate(env):
    if env["use_llvm"]:
        clang.generate(env)
        clangxx.generate(env)
    elif env.use_hot_reload:
        # Critical for proper extension unloading
        env.Append(CXXFLAGS=["-fno-gnu-unique"])

    # Position-independent code for shared libraries
    env.Append(CCFLAGS=["-fPIC", "-Wwrite-strings"])

    # RPATH for finding shared libraries relative to binary
    env.Append(LINKFLAGS=["-Wl,-R,'$$ORIGIN'"])
```

### Architecture-Specific Flags

```python
# tools/linux.py:26-39
if env["arch"] == "x86_64":
    env.Append(CCFLAGS=["-m64", "-march=x86-64"])
    env.Append(LINKFLAGS=["-m64", "-march=x86-64"])
elif env["arch"] == "x86_32":
    env.Append(CCFLAGS=["-m32", "-march=i686"])
    env.Append(LINKFLAGS=["-m32", "-march=i686"])
elif env["arch"] == "arm64":
    env.Append(CCFLAGS=["-march=armv8-a"])
    env.Append(LINKFLAGS=["-march=armv8-a"])
elif env["arch"] == "rv64":
    # RISC-V 64-bit with general extensions and compressed instructions
    env.Append(CCFLAGS=["-march=rv64gc"])
    env.Append(LINKFLAGS=["-march=rv64gc"])
```

### Static Linking for Portability

```python
# tools/linux.py:41-43
if env["use_static_cpp"]:
    # Link C++ runtime statically for distribution
    env.Append(LINKFLAGS=[
        "-static-libgcc",
        "-static-libstdc++"
    ])

env.Append(CPPDEFINES=["LINUX_ENABLED", "UNIX_ENABLED"])
```

## macOS Platform

### Universal Binary Support

```python
# tools/macos.py:51-56
if env["arch"] == "universal":
    # Build fat binary for both Intel and Apple Silicon
    env.Append(LINKFLAGS=["-arch", "x86_64", "-arch", "arm64"])
    env.Append(CCFLAGS=["-arch", "x86_64", "-arch", "arm64"])
else:
    env.Append(LINKFLAGS=["-arch", env["arch"]])
    env.Append(CCFLAGS=["-arch", env["arch"]])
```

### SDK and Deployment Target

```python
# tools/macos.py:58-64
if env["macos_deployment_target"] != "default":
    # Minimum macOS version for compatibility
    env.Append(CCFLAGS=["-mmacosx-version-min=" + env["macos_deployment_target"]])
    env.Append(LINKFLAGS=["-mmacosx-version-min=" + env["macos_deployment_target"]])

if env["macos_sdk_path"]:
    # Custom SDK path (useful for CI/CD)
    env.Append(CCFLAGS=["-isysroot", env["macos_sdk_path"]])
    env.Append(LINKFLAGS=["-isysroot", env["macos_sdk_path"]])
```

### OSXCross Support

```python
# tools/macos.py:32-48
# Cross-compilation from Linux using OSXCross
if has_osxcross():
    root = os.environ.get("OSXCROSS_ROOT", "")

    if env["arch"] == "arm64":
        basecmd = root + "/target/bin/arm64-apple-" + env["osxcross_sdk"] + "-"
    else:
        basecmd = root + "/target/bin/x86_64-apple-" + env["osxcross_sdk"] + "-"

    env["CC"] = basecmd + "clang"
    env["CXX"] = basecmd + "clang++"
    env["AR"] = basecmd + "ar"
    env["RANLIB"] = basecmd + "ranlib"

    # Add OSXCross to PATH for linking
    binpath = os.path.join(root, "target", "bin")
    env.PrependENVPath("PATH", binpath)
```

## Android Platform

### NDK Toolchain Setup

```python
# tools/android.py:30-35
def get_android_ndk_root(env):
    # Prioritize ANDROID_HOME with specific NDK version
    if env["ANDROID_HOME"]:
        return env["ANDROID_HOME"] + "/ndk/" + env["ndk_version"]
    else:
        # Fallback to ANDROID_NDK_ROOT environment variable
        return os.environ.get("ANDROID_NDK_ROOT")
```

### Architecture Configuration

```python
# tools/android.py:77-102
arch_info_table = {
    "arm32": {
        "march": "armv7-a",
        "target": "armv7a-linux-androideabi",
        "compiler_path": "armv7a-linux-androideabi",
        "ccflags": ["-mfpu=neon"],  # NEON SIMD instructions
    },
    "arm64": {
        "march": "armv8-a",
        "target": "aarch64-linux-android",
        "compiler_path": "aarch64-linux-android",
        "ccflags": [],
    },
    "x86_32": {
        "march": "i686",
        "target": "i686-linux-android",
        "compiler_path": "i686-linux-android",
        "ccflags": ["-mstackrealign"],  # Stack alignment for compatibility
    },
    "x86_64": {
        "march": "x86-64",
        "target": "x86_64-linux-android",
        "compiler_path": "x86_64-linux-android",
        "ccflags": [],
    }
}
```

### Toolchain Configuration

```python
# tools/android.py:105-119
# LLVM-based toolchain (required for modern Android)
env["CC"] = toolchain + "/bin/clang"
env["CXX"] = toolchain + "/bin/clang++"
env["LINK"] = toolchain + "/bin/clang++"
env["AR"] = toolchain + "/bin/llvm-ar"
env["AS"] = toolchain + "/bin/llvm-as"
env["STRIP"] = toolchain + "/bin/llvm-strip"
env["RANLIB"] = toolchain + "/bin/llvm-ranlib"

# Target configuration with API level
env.Append(CCFLAGS=[
    "--target=" + arch_info["target"] + env["android_api_level"],
    "-march=" + arch_info["march"],
    "-fPIC"  # Position-independent code required
])
```

### API Level Validation

```python
# tools/android.py:50-53
if int(env["android_api_level"]) < 24:
    print("WARNING: minimum supported Android target api is 24.")
    env["android_api_level"] = "24"
```

## iOS Platform

### Simulator vs Device Configuration

```python
# tools/ios.py:36-43
if env["ios_simulator"]:
    sdk_name = "iphonesimulator"
    env.Append(CCFLAGS=["-mios-simulator-version-min=" + env["ios_min_version"]])
    env.Append(LINKFLAGS=["-mios-simulator-version-min=" + env["ios_min_version"]])
else:
    sdk_name = "iphoneos"
    env.Append(CCFLAGS=["-miphoneos-version-min=" + env["ios_min_version"]])
    env.Append(LINKFLAGS=["-miphoneos-version-min=" + env["ios_min_version"]])
```

### Universal Binary for iOS

```python
# tools/ios.py:86-92
if env["arch"] == "universal":
    if env["ios_simulator"]:
        # Simulator: both x86_64 (Intel Mac) and arm64 (Apple Silicon Mac)
        env.Append(LINKFLAGS=["-arch", "x86_64", "-arch", "arm64"])
        env.Append(CCFLAGS=["-arch", "x86_64", "-arch", "arm64"])
    else:
        # Device: arm64 only (iPhone/iPad)
        env.Append(LINKFLAGS=["-arch", "arm64"])
        env.Append(CCFLAGS=["-arch", "arm64"])
```

### SDK Path Detection

```python
# tools/ios.py:46-54
if env["IOS_SDK_PATH"] == "":
    try:
        # Use xcrun to find SDK
        env["IOS_SDK_PATH"] = subprocess.check_output(
            ["xcrun", "--sdk", sdk_name, "--show-sdk-path"]
        ).strip().decode("utf-8")
    except (subprocess.CalledProcessError, OSError):
        raise ValueError(f"Failed to find SDK path for {sdk_name}")

# Apply SDK settings
env.Append(CCFLAGS=["-isysroot", env["IOS_SDK_PATH"]])
env.Append(LINKFLAGS=["-isysroot", env["IOS_SDK_PATH"]])
```

## Web Platform

### Emscripten Configuration

```python
# tools/web.py:14-19
# Emscripten toolchain
env["CC"] = "emcc"
env["CXX"] = "em++"
env["AR"] = "emar"
env["RANLIB"] = "emranlib"

# tools/web.py:20-23
# Handle long command lines on Windows
env["ARCOM_POSIX"] = env["ARCOM"].replace("$TARGET", "$TARGET.posix")
                                 .replace("$SOURCES", "$SOURCES.posix")
env["ARCOM"] = "${TEMPFILE(ARCOM_POSIX)}"
```

### WebAssembly Configuration

```python
# tools/web.py:33-34
# Output WebAssembly module
env["SHLIBSUFFIX"] = ".wasm"

# tools/web.py:36-39
# Thread support via SharedArrayBuffer
if env["threads"]:
    env.Append(CCFLAGS=["-sUSE_PTHREADS=1"])
    env.Append(LINKFLAGS=["-sUSE_PTHREADS=1"])

# tools/web.py:41-43
# Build as side module (dynamic library)
env.Append(CCFLAGS=["-sSIDE_MODULE=1"])
env.Append(LINKFLAGS=["-sSIDE_MODULE=1"])
```

### WebAssembly Features

```python
# tools/web.py:45-47
# Enable BigInt for 64-bit integer support
# Must match Godot's build configuration
env.Append(LINKFLAGS=["-sWASM_BIGINT"])

# tools/web.py:49-51
# Force WebAssembly-native exception handling
env.Append(CCFLAGS=["-sSUPPORT_LONGJMP='wasm'"])
env.Append(LINKFLAGS=["-sSUPPORT_LONGJMP='wasm'"])
```

## Common Compiler Flags

### C++ Standard and Exceptions

```python
# tools/common_compiler_flags.py:33-47
# C++17 required
if env.get("is_msvc", False):
    env.Append(CXXFLAGS=["/std:c++17"])
else:
    env.Append(CXXFLAGS=["-std=c++17"])

# Disable exceptions for size/performance
if env["disable_exceptions"]:
    if env.get("is_msvc", False):
        env.Append(CPPDEFINES=[("_HAS_EXCEPTIONS", 0)])
    else:
        env.Append(CXXFLAGS=["-fno-exceptions"])
```

### Optimization Levels

```python
# tools/common_compiler_flags.py:113-123
if env["optimize"] == "speed":
    env.Append(CCFLAGS=["-O3"])
elif env["optimize"] == "speed_trace":
    # Better debugging than -O3
    env.Append(CCFLAGS=["-O2"])
elif env["optimize"] == "size":
    env.Append(CCFLAGS=["-Os"])
elif env["optimize"] == "debug":
    env.Append(CCFLAGS=["-Og"])
elif env["optimize"] == "none":
    env.Append(CCFLAGS=["-O0"])
```

### Link-Time Optimization

```python
# tools/common_compiler_flags.py:125-133
if env["lto"] == "thin":
    # ThinLTO (LLVM only)
    if not env["use_llvm"]:
        print("ThinLTO requires LLVM")
        env.Exit(255)
    env.Append(CCFLAGS=["-flto=thin"])
    env.Append(LINKFLAGS=["-flto=thin"])
elif env["lto"] == "full":
    # Full LTO
    env.Append(CCFLAGS=["-flto"])
    env.Append(LINKFLAGS=["-flto"])
```

### Debug Symbols

```python
# tools/common_compiler_flags.py:92-105
if env["debug_symbols"]:
    # DWARF-4 for better compatibility
    env.Append(CCFLAGS=["-gdwarf-4"])

    if using_emcc(env):
        # Emscripten needs -g3 for full debug info
        env.AppendUnique(CCFLAGS=["-g3"])
        env.AppendUnique(LINKFLAGS=["-gdwarf-4", "-g3"])
    elif env.dev_build:
        env.Append(CCFLAGS=["-g3"])  # Maximum debug info
    else:
        env.Append(CCFLAGS=["-g2"])  # Standard debug info
```

### Symbol Visibility

```python
# tools/common_compiler_flags.py:49-55
if env["symbols_visibility"] == "visible":
    env.Append(CCFLAGS=["-fvisibility=default"])
    env.Append(LINKFLAGS=["-fvisibility=default"])
elif env["symbols_visibility"] == "hidden":
    # Hide symbols by default (reduces binary size)
    env.Append(CCFLAGS=["-fvisibility=hidden"])
    env.Append(LINKFLAGS=["-fvisibility=hidden"])
```

## Cross-Compilation

### Host vs Target Detection

```python
# Platform detection logic
def detect_cross_compilation(env):
    host_platform = platform.system().lower()
    target_platform = env["platform"]

    is_cross_compile = host_platform != target_platform

    if is_cross_compile:
        print(f"Cross-compiling from {host_platform} to {target_platform}")
        setup_cross_toolchain(env, target_platform)
```

### Toolchain Path Management

```python
# Common pattern for cross-compilation toolchains
def setup_cross_toolchain(env, target):
    toolchain_root = get_toolchain_root(target)

    # Prepend toolchain to PATH
    env.PrependENVPath("PATH", os.path.join(toolchain_root, "bin"))

    # Set compiler prefix
    prefix = get_target_triple(target)
    env["CC"] = prefix + "-gcc"
    env["CXX"] = prefix + "-g++"
    env["AR"] = prefix + "-ar"
    env["RANLIB"] = prefix + "-ranlib"
```

### SDK Management

```python
# SDK location patterns
sdk_paths = {
    "android": "$ANDROID_HOME/ndk/$version",
    "ios": "xcrun --sdk $sdk --show-sdk-path",
    "macos": "$OSXCROSS_ROOT/target/SDK",
    "windows": "$MINGW_PREFIX",
}

# Sysroot configuration
env.Append(CCFLAGS=["-isysroot", sdk_path])
env.Append(LINKFLAGS=["-isysroot", sdk_path])
```

## Platform-Specific Defines

Each platform sets specific preprocessor defines:

```python
platform_defines = {
    "windows": ["WINDOWS_ENABLED", "TYPED_METHOD_BIND", "NOMINMAX"],
    "linux": ["LINUX_ENABLED", "UNIX_ENABLED"],
    "macos": ["MACOS_ENABLED", "UNIX_ENABLED"],
    "android": ["ANDROID_ENABLED", "UNIX_ENABLED"],
    "ios": ["IOS_ENABLED", "UNIX_ENABLED"],
    "web": ["WEB_ENABLED", "UNIX_ENABLED"],
}
```

## Platform-Specific Libraries

### Runtime Library Linking

```python
# Windows: Static vs Dynamic CRT
if static:
    msvc_flags = ["/MT"]  # Static runtime
    mingw_flags = ["-static", "-static-libgcc", "-static-libstdc++"]
else:
    msvc_flags = ["/MD"]  # Dynamic runtime
    mingw_flags = []  # Dynamic by default

# Linux/macOS: Static libstdc++ for portability
if static:
    unix_flags = ["-static-libgcc", "-static-libstdc++"]
```

### Platform Library Suffixes

```python
library_suffixes = {
    "windows": {
        "static": ".lib" if msvc else ".a",
        "shared": ".dll",
        "import": ".lib" if msvc else ".a"
    },
    "linux": {
        "static": ".a",
        "shared": ".so"
    },
    "macos": {
        "static": ".a",
        "shared": ".dylib"
    },
    "android": {
        "static": ".a",
        "shared": ".so"
    },
    "ios": {
        "static": ".a",
        "shared": ".dylib"  # Frameworks use .framework
    },
    "web": {
        "static": ".a",
        "shared": ".wasm"
    }
}
```

## Build Output Organization

```python
# Platform-specific output paths
def get_output_path(env):
    # Binary naming: libgodot-cpp.{platform}.{target}.{arch}{suffix}
    platform = env["platform"]
    target = env["target"]
    arch = env["arch"]

    suffix = f".{platform}.{target}"

    if env.dev_build:
        suffix += ".dev"
    if env["precision"] == "double":
        suffix += ".double"

    suffix += f".{arch}"

    if env.get("ios_simulator"):
        suffix += ".simulator"

    return f"bin/libgodot-cpp{suffix}{env['LIBSUFFIX']}"
```

## Summary

The platform-specific implementation in godot-cpp demonstrates sophisticated cross-platform build management:

1. **Compiler Abstraction**: Unified interface across MSVC, GCC, Clang, Emscripten
2. **Architecture Support**: x86, x64, ARM, ARM64, RISC-V, WebAssembly
3. **SDK Integration**: Automatic detection and configuration of platform SDKs
4. **Cross-Compilation**: Full support for building from any host to any target
5. **Runtime Libraries**: Flexible static/dynamic linking configurations
6. **Platform Features**: Thread support, SIMD instructions, debug symbols
7. **Optimization**: Platform-specific optimization defaults and LTO support

Each platform implementation carefully balances performance, compatibility, and ease of use while maintaining consistent behavior across all supported targets.

*Supported Platforms: 6 (Windows, Linux, macOS, Android, iOS, Web)*
*Supported Architectures: 9+ per platform*
*Compiler Support: MSVC, GCC, Clang, MinGW, Emscripten*
*Cross-compilation: Full matrix of host/target combinations*
