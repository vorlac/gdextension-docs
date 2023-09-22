# GDExtension Overview

The functionality behind godot's GDExtension feature allows for native development through the implemention of a dynamic library that can add custom game logic functionality as well as extending engine functionality.

The implementaion of this feature works through special libraries that generate bindings to interface the dynamically library (.so/.dll) with the engine, which in turn allows the engine to expose the custom functionality implemented in the library to the editor and GDScript.

This means you can create custom nodes (with custom properties, signals, etc), and add them to a scene through the editor. Or levereage them in gdscript the same way you'd use any built-in node type.

## Example Use Cases

* Leveraging C++, rust, D, or any other language that provides a bindings library to implement some performance sensitive game logic.
* Adding support for any 3rd party libraries by linking them to your GDExtension library.
* Implementing custom node types, resources, singletons, or low level servers to exend the functionality that's offered through Godot's built-in nodes, resources, singletons, servers, etc.
* Implementing an entire game as a GDExtension through the functionality exposed by the bindings.

## Godot Bindings Libraries

The only officially supported bindings library is godot-cpp, allowing for GDExtensions to be implemented in C++, however there are a handful of additional community supporeted bindings for other languages.

1. [godot-cpp](https://github.com/godotengine/godot-cpp)
1. [godot-rust](https://github.com/godot-rust/godot-rust)
1. [godot-d](https://github.com/godot-d/godot-d)

# Bindings Overview




# Core Native Functionality
