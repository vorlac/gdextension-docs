# GDExtension Overview

The functionality behind godot's GDExtension feature allows for native development through the implemention of a dynamic library that can add custom game logic functionality as well as extending engine functionality.

The implementaion of this feature works through special libraries that generate bindings to interface the dynamically library (.so/.dll) with the engine, which in turn allows the engine to expose the custom functionality implemented in the library to the editor and GDScript.

This means you can create custom nodes (with custom properties, signals, etc), and add them to a scene through the editor. Or leverage them in gdscript the same way you'd use any built-in node type.

`TODO: just reference existing GDExtension docs page instead of putting stuff like this in here?`

## Example Use Cases

* Leveraging C++, rust, D, or any other language that provides a bindings library to implement some performance sensitive game logic.
* Adding support for any 3rd party libraries by linking them to your GDExtension library.
* Implementing custom node types, resources, singletons, or low level servers to exend the functionality that's offered through Godot's built-in nodes, resources, singletons, servers, etc.
* Implementing an entire game as a GDExtension through the functionality exposed by the bindings.

`TODO: review/expand list above`

`TODO: just reference existing GDExtension docs page instead of putting stuff like this in here?`

## Godot Bindings Libraries

The only officially supported bindings library is godot-cpp, allowing for GDExtensions to be implemented in C++, however there are a handful of additional community supporeted bindings for other languages.

1. [godot-cpp](https://github.com/godotengine/godot-cpp)
1. [godot-rust](https://github.com/godot-rust/godot-rust)
1. [godot-d](https://github.com/godot-d/godot-d)

`TODO: exclude mention of other bindings project in Godot docs since godot-cpp is the only one officially supported?`

# Bindings Overview

* Brief overview of the bindings library
    * high level description of how it works
        * visualization / flow diagram
    * `static void _bind_methods()` importance
        * its purpose
        * when it's necessary to use
    * `ClassDB`
    * macros
        * `GDCLASS` macro
        * Variant type conversion template specialization macros
    * GDExtension library initialization levels
        * When each should be used (Core, Server, Scene, Editor)
* How/when Variant compatible objects are necessary
* How/when safe to use STL data structures, 3rd party libraries, and mix external data structures with internal Godot classes.

`TODO: anything big missing above?`

# Core Native Functionality

## Core Godot Engine Interfaces

`TODO: add basic explanation of what each of the following are (check existing docs to link to before writing anything that isn't specific to native development)`

`TODO: add basic example implementations for each object type below`

### Servers
### Singletons
* list of headers/singletons commonly used to leverage engine, scene, editor, etc functionality
### Nodes
### Editor Plugins

# Native Development Basics

`TODO: any major topics missing below?`

## General Utilities

* `UtilityFunctions` namespace overview
* Print to terminal / editor console
* useful debug macros
* console redirection?
    * `TODO: not sure is this is easy to override based on what's exposed in the bindings after messing around with this a bit myself`
* Drawing debug visualizations using `_draw`
    * `TODO: ^ might be specific to 2D rendering`
    * `TODO: move elsewhere?`
* etc..

## Memory Management

When working with native code in a GDExtension project, memory needs to be managed in a few specific ways depending on the situation.

`TODO: should the following be organized by the allocation functions rather than grouped by type of object instead?`

## Resource / Scene Managements

* Loading resources from disk
* Loading packed scenes from disk

## Signals

* How custom signals are defined in native code (`TODO too much overlap with existing docs page / GDExtension example?`)
    * Signal bindings / declarations in native code
    * Signal connections in native code / using `godot::Callable`
    * emitting signals in native code
* How existing signals are defined (`TODO too much overlap with existing docs page / GDExtension example?`)

## Notifications

* Notifaction propigation rules
    * how to adjust/override signal propigation in scene tree hierarchy.
* how/when to use/define `void notification(int what)` in custom `GDCLASS` nodes

## Method Overrides

* Expected behavior when overriding native methods in native code
* Expected behavior when overriding native methods in gdscript

## GDCLASS Inheritance
* how to structure inheritance for custom `GDCLASS` definitions.

## Memory Management

`TODO: go into some detail about when/how to use some of the following... i.e. sentence below`

`Resource` instantiation (along with any objects derived from the Resource class) use reference counted objects.

```cpp
Ref<Resource> res = memnew(Resource);
// OR
Ref<Resource> res;
res.instantiate();
```

`TODO: worth just exluding most of these aside from memnew(), Ref<>, and node->queue_free()?`

```cpp
    memalloc()
    memrealloc()
    memfree()

    memnew()
    memnew_arr()
    memdelete()
    memdelete_arr()

    memnew_allocator()
    memnew_placement()

    node->queue_free()

    Ref<Resource>
```

## Project Structure

#### File/Folder strucutre
* outline recommended file/folder/repo structure for GDExtension project

#### Build systems
* pros/cons of scons vs cmake vs ???

#### IDE Support / Integration
* How to configure CMake for best IDE integration
* Example sconstruct and cmakelists.txt/cmakepresets.json templates
* Debugging
    * Native debugging for GDExtension library
    * Native debugging for the engine alongside GDExtension library
    * Example platform agnostic launch configuration files for VSCode / Visual Studio / CLion / etc

#### 3rd Party Libraries
* how/where to integrate external native 3rd party libs
    * crossplatform package managers (vcpkg / conan)
    * building from source manually
    * cmake & scons examples
* examples using scons and cmake

## Overriding/Replacing Godot Built-in Functionality
* Haven't tried this out myself, and don't know if it's possible to do yet (or even intended), especially leveraging GDExtension.
* Example might be how to approach replacing godot physics/rendering with external native physics/rendering libraries.
`TODO does this make sense in Godot docs??`

## Code Examples

`TODO worth including example snippets of code that covers everything above in a separate section like this? or keep stuff like that in-line with the above sections?`
