# Build System Architecture Internal Documentation

## Table of Contents
1. [Overview](#overview)
2. [SCons Build System](#scons-build-system)
3. [Platform Detection & Configuration](#platform-detection--configuration)
4. [Compiler Configuration](#compiler-configuration)
5. [Code Generation Pipeline](#code-generation-pipeline)
6. [Optimization Strategies](#optimization-strategies)
7. [Build Profiles](#build-profiles)
8. [Platform-Specific Toolchains](#platform-specific-toolchains)

## Overview

**The sophisticated system that turns C++ code into working GDExtension libraries:** Building godot-cpp is more complex than typical C++ projects because it must work across multiple platforms, generate thousands of lines of binding code automatically, and integrate with Godot's specific build requirements. The build system orchestrates code generation, cross-compilation, optimization, and packaging into a single streamlined process.

**Why the build system is complex:** godot-cpp doesn't just compile your C++ code - it first generates most of that C++ code from Godot's API definition. It must handle platform-specific compiler quirks, link against platform-specific libraries, optimize for different targets (debug vs. release vs. editor), and produce libraries that work with Godot's exact ABI requirements. This complexity is hidden behind simple commands like `scons platform=windows`.

**What happens during a build:** The process starts by detecting your platform and parsing build options, then generates C++ bindings from Godot's API definition, configures compilers with the right flags and optimizations for your target platform, compiles thousands of generated source files plus your extension code, and finally links everything into libraries that Godot can load.

godot-cpp uses SCons as its primary build system, with CMake as a secondary option. The build system handles cross-platform compilation, automatic code generation, optimization profiles, and platform-specific toolchain configuration.

### Build System Architecture
```
SConstruct (Entry Point)
    ↓
tools/godotcpp.py (Main Configuration)
    ↓
Platform Tools (windows.py, linux.py, etc.)
    ↓
┌─────────────────────────────────┐
│ Code Generation Pipeline        │
│ - binding_generator.py          │
│ - build_profile.py              │
│ - doc_source_generator.py       │
└─────────────────────────────────┘
    ↓
Compilation & Linking
```

## SCons Build System

### Entry Point (SConstruct)

The build starts with the minimal SConstruct file:

```python
# SConstruct:10-11
EnsureSConsVersion(4, 0)
EnsurePythonVersion(3, 8)

# SConstruct:17-18
# Default environment with no platform defaults
env = Environment(tools=["default"], PLATFORM="")

# SConstruct:35-37
# Load godotcpp tool
cpp_tool = Tool("godotcpp", toolpath=[Dir("tools").srcnode().abspath])
cpp_tool.options(opts, env)  # Configure options
cpp_tool.generate(env)       # Generate build configuration

# SConstruct:54
library = env.GodotCPP()  # Build the library
```

### Build Options Configuration

The build system provides extensive configuration options:

```python
# tools/godotcpp.py:207-387
def options(opts, env):
    # Platform detection
    if sys.platform.startswith("linux"):
        default_platform = "linux"
    elif sys.platform == "darwin":
        default_platform = "macos"
    elif sys.platform == "win32" or sys.platform == "msys":
        default_platform = "windows"

    # Core build options
    opts.Add(EnumVariable(
        key="platform",
        help="Target platform",
        default=default_platform,
        allowed_values=["linux", "macos", "windows", "android", "ios", "web"]
    ))

    opts.Add(EnumVariable(
        key="target",
        help="Compilation target",
        default="template_debug",
        allowed_values=("editor", "template_release", "template_debug")
    ))

    opts.Add(EnumVariable(
        key="arch",
        help="CPU architecture",
        default="",  # Auto-detect
        allowed_values=["", "universal", "x86_32", "x86_64",
                       "arm32", "arm64", "rv64", "ppc32", "ppc64", "wasm32"]
    ))
```

### Build Variables and Paths

```python
# tools/godotcpp.py:30-52
def normalize_path(val, env):
    """Normalize user-provided paths to absolute paths"""
    return os.path.join(env.Dir("#").abspath, val)

def validate_file(key, val, env):
    """Validate that a file exists"""
    if not os.path.isfile(normalize_path(val, env)):
        raise UserError("'%s' is not a file: %s" % (key, val))

def validate_dir(key, val, env):
    """Validate that a directory exists"""
    if not os.path.isdir(normalize_path(val, env)):
        raise UserError("'%s' is not a directory: %s" % (key, val))
```

## Platform Detection & Configuration

**Platform Configuration System**: The build system automatically detects your development environment and target platform, then configures compiler flags, toolchain settings, and library paths accordingly. This system handles cross-compilation scenarios, architecture-specific optimizations, and platform API differences transparently.

### Architecture Detection

The build system implements sophisticated architecture detection:

```python
# tools/godotcpp.py:175-200
architecture_array = [
    "", "universal", "x86_32", "x86_64",
    "arm32", "arm64", "rv64", "ppc32", "ppc64", "wasm32"
]

architecture_aliases = {
    "x64": "x86_64",
    "amd64": "x86_64",
    "armv7": "arm32",
    "armv8": "arm64",
    "arm64v8": "arm64",
    "aarch64": "arm64",
    "rv": "rv64",
    "riscv": "rv64",
    "riscv64": "rv64",
    "ppcle": "ppc32",
    "ppc": "ppc32",
    "ppc64le": "ppc64"
}

# tools/godotcpp.py:410-432
def generate(env):
    if env["arch"] == "":
        # Platform-specific defaults
        if env["platform"] in ["macos", "ios"]:
            env["arch"] = "universal"  # Fat binary
        elif env["platform"] == "android":
            env["arch"] = "arm64"
        elif env["platform"] == "web":
            env["arch"] = "wasm32"
        else:
            # Auto-detect host architecture
            host_machine = platform.machine().lower()
            if host_machine in architecture_array:
                env["arch"] = host_machine
            elif host_machine in architecture_aliases.keys():
                env["arch"] = architecture_aliases[host_machine]
            elif "86" in host_machine:
                # Catches x86, i386, i486, i586, i686, etc.
                env["arch"] = "x86_32"
```

### Platform Tool Loading

Each platform has a specific tool module:

```python
# tools/godotcpp.py:462-467
tool = Tool(env["platform"], toolpath=get_platform_tools_paths(env))

if tool is None or not tool.exists(env):
    raise ValueError("Required toolchain not found for platform " + env["platform"])

tool.generate(env)  # Platform-specific configuration
```

## Compiler Configuration

### Compiler Flags Management

The build system manages complex compiler flag combinations:

```python
# tools/godotcpp.py:441-460
# Optimization levels based on build target
if env.dev_build:
    opt_level = "none"
elif env.debug_features:
    opt_level = "speed_trace"  # Optimized with debug info
else:  # Release
    opt_level = "speed"

env["optimize"] = ARGUMENTS.get("optimize", opt_level)
env["debug_symbols"] = get_cmdline_bool("debug_symbols", env.dev_build)

# tools/godotcpp.py:469-503
# Conditional compilation defines
if env["threads"]:
    env.Append(CPPDEFINES=["THREADS_ENABLED"])

if env.use_hot_reload:
    env.Append(CPPDEFINES=["HOT_RELOAD_ENABLED"])

if env.editor_build:
    env.Append(CPPDEFINES=["TOOLS_ENABLED"])

if env.debug_features:
    env.Append(CPPDEFINES=["DEBUG_ENABLED"])

if env.dev_build:
    env.Append(CPPDEFINES=["DEV_ENABLED"])
else:
    env.Append(CPPDEFINES=["NDEBUG"])  # Disable assert()

if env["precision"] == "double":
    env.Append(CPPDEFINES=["REAL_T_IS_DOUBLE"])

env.Append(CPPDEFINES=["GDEXTENSION"])  # Always defined
```

### Windows-Specific Compiler Configuration

```python
# tools/windows.py:90-136
def generate(env):
    if not env["use_mingw"] and msvc.exists(env):
        # MSVC Configuration
        if env["arch"] == "x86_64":
            env["TARGET_ARCH"] = "amd64"
        elif env["arch"] == "arm64":
            env["TARGET_ARCH"] = "arm64"

        env["is_msvc"] = True
        msvc.generate(env)

        env.Append(CPPDEFINES=["TYPED_METHOD_BIND", "NOMINMAX"])
        env.Append(CCFLAGS=["/utf-8"])  # UTF-8 source files
        env.Append(LINKFLAGS=["/WX"])   # Treat linker warnings as errors

        if env["use_llvm"]:
            env["CC"] = "clang-cl"
            env["CXX"] = "clang-cl"

        # Runtime library selection
        if env["debug_crt"]:
            env.AppendUnique(CCFLAGS=["/MDd"])  # Debug DLL runtime
        else:
            if env["use_static_cpp"]:
                env.AppendUnique(CCFLAGS=["/MT"])  # Static runtime
            else:
                env.AppendUnique(CCFLAGS=["/MD"])  # DLL runtime
```

### MinGW Configuration

```python
# tools/windows.py:162-200
# Cross-compilation using MinGW
if env["arch"] == "x86_64":
    prefix += "x86_64"
elif env["arch"] == "arm64":
    prefix += "aarch64"
elif env["arch"] == "arm32":
    prefix += "armv7"
elif env["arch"] == "x86_32":
    prefix += "i686"

if env["use_llvm"]:
    env["CXX"] = prefix + "-w64-mingw32-clang++"
    env["CC"] = prefix + "-w64-mingw32-clang"
    env["AR"] = prefix + "-w64-mingw32-llvm-ar"
else:
    env["CXX"] = prefix + "-w64-mingw32-g++"
    env["CC"] = prefix + "-w64-mingw32-gcc"
    env["AR"] = prefix + "-w64-mingw32-gcc-ar"

env.Append(LINKFLAGS=["-Wl,--no-undefined"])
if env["use_static_cpp"]:
    env.Append(LINKFLAGS=[
        "-static",
        "-static-libgcc",
        "-static-libstdc++"
    ])
```

## Code Generation Pipeline

### Binding Generation Build Process

The build system integrates code generation seamlessly:

```python
# tools/godotcpp.py:135-170
def scons_emit_files(target, source, env):
    """Emit all files that will be generated"""
    profile_filepath = env.get("build_profile", "")

    # Generate trimmed API based on profile
    api = generate_trimmed_api(str(source[0]), profile_filepath)

    # Get list of files that will be generated
    files = []
    for f in _get_file_list(api, target[0].abspath, True, True):
        file = env.File(f)
        if profile_filepath:
            env.Depends(file, profile_filepath)  # Rebuild if profile changes
        files.append(file)

    env["godot_cpp_gen_dir"] = target[0].abspath
    return files, source

def scons_generate_bindings(target, source, env):
    """Actually generate the bindings"""
    api = generate_trimmed_api(str(source[0]), profile_filepath)

    _generate_bindings(
        api,
        str(source[0]),  # extension_api.json
        env["generate_template_get_node"],
        "32" if "32" in env["arch"] else "64",
        env["precision"],
        env["godot_cpp_gen_dir"]
    )
```

### Builder Registration

```python
# tools/godotcpp.py:528-534
env.Append(
    BUILDERS={
        "GodotCPPBindings": Builder(
            action=Action(scons_generate_bindings, "$GENCOMSTR"),
            emitter=scons_emit_files
        ),
        "GodotCPPDocData": Builder(action=scons_generate_doc_source)
    }
)
```

### Source File Collection

```python
# tools/godotcpp.py:556-563
def _godot_cpp(env):
    # Sources to compile
    sources = [
        *env.Glob("src/*.cpp"),
        *env.Glob("src/classes/*.cpp"),
        *env.Glob("src/core/*.cpp"),
        *env.Glob("src/variant/*.cpp"),
        *tuple(f for f in bindings if str(f).endswith(".cpp")),  # Generated
    ]
```

## Optimization Strategies

### Build Parallelization

```python
# tools/godotcpp.py:391-407
# Auto-detect CPU cores for parallel compilation
initial_num_jobs = env.GetOption("num_jobs")
altered_num_jobs = initial_num_jobs + 1
env.SetOption("num_jobs", altered_num_jobs)

if env.GetOption("num_jobs") == altered_num_jobs:
    cpu_count = os.cpu_count()
    if cpu_count is not None:
        # Leave one core free for system responsiveness
        safer_cpu_count = cpu_count if cpu_count <= 4 else cpu_count - 1
        print(f"Auto-detected {cpu_count} CPU cores. Using {safer_cpu_count} cores.")
        env.SetOption("num_jobs", safer_cpu_count)
```

### Link-Time Optimization

```python
# tools/godotcpp.py:370-376
opts.Add(EnumVariable(
    "lto",
    "Link-time optimization",
    "none",
    ("none", "auto", "thin", "full")
))

# Platform tools apply LTO flags based on this setting
# Example: -flto=thin for Clang, -flto for GCC
```

### Build Caching

```python
# SConstruct:48-51
scons_cache_path = os.environ.get("SCONS_CACHE")
if scons_cache_path is not None:
    CacheDir(scons_cache_path)  # Enable build caching
    Decider("MD5")  # Use MD5 for change detection
```

## Build Profiles

### Profile System Architecture

Build profiles allow selective compilation of API subsets:

```python
# build_profile.py:5-64
def parse_build_profile(profile_filepath, api):
    """Parse build profile JSON to determine included/excluded classes"""
    with open(profile_filepath, encoding="utf-8") as profile_file:
        profile = json.load(profile_file)

    # Build parent-child relationships
    parents = {}
    children = {}
    for engine_class in api["classes"]:
        parent = engine_class.get("inherits", "")
        child = engine_class["name"]
        parents[child] = parent
        if parent:
            children[parent] = children.get(parent, [])
            children[parent].append(child)

    # Process enabled_classes (whitelist mode)
    included = []
    front = list(profile.get("enabled_classes", []))
    if front:
        # Essential classes always included
        front.append("WorkerThreadPool")
        front.append("ClassDB")
        front.append("ClassDBSingleton")
        front.append("FileAccess")
        front.append("Image")
        front.append("XMLParser")
        front.append("Semaphore")

    while front:
        cls = front.pop()
        if cls not in included:
            included.append(cls)
            # Include parent classes recursively
            parent = parents.get(cls, "")
            if parent:
                front.append(parent)

    # Process disabled_classes (blacklist mode)
    excluded = []
    front = list(profile.get("disabled_classes", []))
    while front:
        cls = front.pop()
        if cls not in excluded:
            excluded.append(cls)
            # Exclude child classes recursively
            front += children.get(cls, [])
```

### Profile-Based API Trimming

```python
# build_profile.py:67-98
def generate_trimmed_api(source_api_filepath, profile_filepath):
    """Generate trimmed API based on build profile"""
    with open(source_api_filepath, encoding="utf-8") as api_file:
        api = json.load(api_file)

    if profile_filepath == "":
        return api  # No trimming

    build_profile = parse_build_profile(profile_filepath, api)

    # Filter classes
    classes = []
    for class_api in api["classes"]:
        if not is_class_included(class_api["name"], build_profile):
            continue

        # Filter methods
        if "methods" in class_api:
            methods = []
            for method in class_api["methods"]:
                if not is_method_included(method, build_profile, engine_classes):
                    continue
                methods.append(method)
            class_api["methods"] = methods

        classes.append(class_api)

    api["classes"] = classes
    return api
```

## Platform-Specific Toolchains

### Windows Platform Tools

```python
# tools/windows.py:10-73
def silence_msvc(env):
    """Reduce MSVC compiler output noise"""
    # Capture and filter MSVC output
    # Only show actual errors/warnings, not file names being compiled

    old_spawn = env["SPAWN"]

    def spawn_capture(sh, escape, cmd, args, env):
        if args[0] not in ["cl", "link"]:
            return old_spawn(sh, escape, cmd, args, env)

        # Redirect stdout to temp file, filter, then output
        tmp_stdout, tmp_stdout_name = tempfile.mkstemp()
        args.append(f">{tmp_stdout_name}")
        ret = old_spawn(sh, escape, cmd, args, env)

        # Process captured output, filtering noise
        # ...

    env["SPAWN"] = spawn_capture
```

### Cross-Compilation Support

```python
# Platform detection for cross-compilation
def detect_cross_compile_target(env):
    """Detect cross-compilation scenarios"""
    host_platform = platform.system().lower()
    target_platform = env["platform"]

    if host_platform != target_platform:
        # Cross-compiling
        if target_platform == "android":
            # Android NDK toolchain
            setup_android_toolchain(env)
        elif target_platform == "ios":
            # iOS SDK toolchain
            setup_ios_toolchain(env)
        elif target_platform == "web":
            # Emscripten toolchain
            setup_emscripten_toolchain(env)
```

### Universal Binary Support (macOS/iOS)

```python
# Universal binary for Apple platforms
if env["arch"] == "universal":
    if env["platform"] == "macos":
        # Compile for both x86_64 and arm64
        env.Append(CCFLAGS=["-arch", "x86_64", "-arch", "arm64"])
        env.Append(LINKFLAGS=["-arch", "x86_64", "-arch", "arm64"])
    elif env["platform"] == "ios":
        # Device (arm64) and simulator (x86_64/arm64)
        # Handled by separate builds and lipo
```

## Build System Integration

### Library Building

```python
# tools/godotcpp.py:574-589
def _godot_cpp(env):
    library_name = "libgodot-cpp" + env["suffix"] + env["LIBSUFFIX"]

    if env["build_library"]:
        library = env.StaticLibrary(
            target=env.File("bin/%s" % library_name),
            source=sources
        )
        env.NoCache(library)  # Always rebuild library

        default_args = [library]

        # Add compile_commands.json generation
        if env.get("compiledb", False):
            default_args += ["compiledb"]

        env.Default(*default_args)

    env.AppendUnique(LIBS=[env.File("bin/%s" % library_name)])
    return library
```

### Build Suffix Generation

```python
# tools/godotcpp.py:504-517
# Generate unique suffix for build artifacts
suffix = ".{}.{}".format(env["platform"], env["target"])

if env.dev_build:
    suffix += ".dev"
if env["precision"] == "double":
    suffix += ".double"

suffix += "." + env["arch"]

if env["ios_simulator"]:
    suffix += ".simulator"
if not env["threads"]:
    suffix += ".nothreads"

env["suffix"] = suffix
env["OBJSUFFIX"] = suffix + env["OBJSUFFIX"]  # Unique object files
```

## Compilation Database Support

```python
# tools/godotcpp.py:519-521
# Generate compile_commands.json for IDE integration
env.Tool("compilation_db")
env.Alias("compiledb",
          env.CompilationDatabase(normalize_path(env["compiledb_file"], env)))
```

## Summary

The godot-cpp build system represents a sophisticated cross-platform compilation framework:

1. **Platform Abstraction**: Unified interface across Windows, Linux, macOS, Android, iOS, Web
2. **Auto-Detection**: CPU architecture, compiler availability, optimal parallelization
3. **Code Generation**: Seamless integration of binding generation into build process
4. **Optimization**: Profile-guided optimization, LTO support, build caching
5. **Flexibility**: Build profiles for size optimization, hot reload support, custom toolchains
6. **Developer Experience**: Colored output, compilation database, verbose/quiet modes

The build system ensures consistent, optimized builds across all supported platforms while maintaining flexibility for custom configurations and cross-compilation scenarios.

*Configuration Options: 25+ build variables*
*Supported Platforms: 6 major platforms + custom*
*Supported Architectures: 9 architectures + aliases*
*Build Targets: 3 (editor, template_debug, template_release)*
