# Glossary

## A

**`Abstract Class`**: A class registered with `GDREGISTER_ABSTRACT_CLASS()` that cannot be instantiated directly but serves as a base for other classes. Used for defining interfaces and common functionality.

**`API Surface`**: The public interface exposed by GDExtension libraries to the Godot engine, including methods, properties, and signals that can be called from GDScript or other extensions.

**`Autoload`**: A singleton pattern in Godot where nodes are automatically instantiated when the project starts and remain accessible globally throughout the application lifecycle.

## B

**`Binding`**: The automatically generated C++ code that bridges the gap between Godot's internal C API and C++ classes, providing type-safe wrappers around engine functionality.

**`Binding Generator`**: The Python script (`binding_generator.py`) that reads Godot's extension API definition and generates C++ header and source files for all engine classes.

**`Binary Interface`**: The low-level C interface between Godot engine and GDExtension libraries, defined by function pointers and data structures that enable cross-language communication.

## C

**`ClassDB`**: Godot's internal class registration system that maintains metadata about all registered classes, including their methods, properties, signals, and inheritance relationships.

**`Cross-boundary Call`**: Function calls that cross the boundary between the Godot engine and GDExtension libraries, typically involving data marshalling and type conversion.

**`CustomCallable`**: A wrapper class that allows C++ functions, lambdas, and std::function objects to be used as Godot Callable objects for signal connections and deferred calls.

## D

**`Data Marshalling`**: The process of converting data between different representations when crossing the engine-extension boundary, such as converting C++ types to Variant and back.

## E

**`Extension API`**: The JSON file (`extension_api.json`) that defines all classes, methods, properties, and other API elements available in the Godot engine for extension development.

**`Extension Library`**: A compiled dynamic library (.dll, .so, .dylib) containing GDExtension code that can be loaded by Godot at runtime.

## F

**`Function Pointer Table`**: A structure containing pointers to all available engine functions, passed to extensions during initialization to enable calling engine functionality.

## G

**`GDCLASS`**: A macro used to declare C++ classes that inherit from Godot classes, enabling proper integration with Godot's object system and ClassDB registration.

**`GDExtension`**: Godot's system for creating native extensions using C++ or other languages, replacing the older GDNative system with improved performance and capabilities.

**`GDREGISTER_CLASS`**: A macro used to register standard C++ classes with Godot's ClassDB, making them available for instantiation and use within the engine.

**`GDREGISTER_VIRTUAL_CLASS`**: A macro for registering classes that serve as base classes but may not be directly instantiated, used for polymorphic hierarchies.

**`GDScript`**: Godot's built-in scripting language, designed specifically for game development with Python-like syntax and tight engine integration.

## I

**`Interface Header`**: The C header file (`gdextension_interface.h`) that defines the binary interface between Godot and extensions, including function signatures and data structures.

**`Initialization Function`**: The entry point function that extensions must export, called by Godot during library loading to set up the extension and register classes.

## M

**`Method Binding`**: The process of exposing C++ class methods to the Godot engine so they can be called from GDScript or other parts of the engine.

**`Module Initialization`**: The phase during extension loading where classes are registered with ClassDB and other setup tasks are performed.

## N

**`Notification System`**: Godot's mechanism for broadcasting events (like READY, PROCESS, PHYSICS_PROCESS) to nodes in the scene tree, allowing them to respond to engine state changes.

## O

**`Object System`**: Godot's fundamental architecture where all engine entities inherit from Object, providing reference counting, signals, properties, and other core functionality.

## P

**`Property Binding`**: The mechanism that exposes C++ class member variables as Godot properties, enabling access from the editor and GDScript with automatic getter/setter generation.

## R

**`RefCounted`**: A base class for objects that use automatic memory management through reference counting, ensuring proper cleanup when no longer referenced.

**`Reference Counting`**: An automatic memory management technique where objects track how many references point to them and delete themselves when the count reaches zero.

## S

**`Signal Binding`**: The process of exposing C++ signals to Godot's signal system, allowing connections between objects and event-driven communication patterns.

**`Singleton`**: A design pattern ensuring only one instance of a class exists, commonly used in Godot for engine subsystems like Input, ResourceLoader, and custom autoloads.

## T

**`Thread Safety`**: The property of code that can be safely executed concurrently by multiple threads without causing data races or corruption.

**`Type System`**: The framework that manages type information, conversions, and compatibility between C++ types and Godot's internal type system.

## V

**`Variant`**: Godot's universal value type that can hold any supported data type, used extensively for cross-boundary communication and dynamic typing.

**`Virtual Class`**: A class registered with `GDREGISTER_VIRTUAL_CLASS()` that provides base functionality for derived classes, often used in plugin architectures and polymorphic designs.

**`Virtual Method`**: A C++ method that can be overridden in derived classes, with special handling in GDExtension to ensure proper integration with Godot's object system.

## W

**`Wrapper Class`**: Generated C++ classes that provide a convenient interface around Godot's C API, handling memory management, type conversion, and error checking automatically.
