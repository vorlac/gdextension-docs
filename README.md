# Godot GDExtension Unofficial Technical Documentation

This comprehensive technical documentation provides deep insights into godot-cpp, the C++ bindings for Godot Engine's GDExtension API. The documentation covers everything from getting started to advanced internal implementation details.

## **Introductory Topics**
*For developers looking to start at the beginngin to learn the basics of GDExtension*
- [Quick Start](./getting_started/quick-start.md) - Installation and setup
- [First Extension](https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/gdextension_cpp_example.html) - Godot's official GDExtension introductory project guide
- [Basic Concepts](./getting_started/basic-concepts.md) - Core concepts explained

## **Getting Started**
*Setup guide and overview of GDExtension's architecture*
- [Quick Start Guide](./getting_started/quick-start.md) - Get up and running quickly
- [First Extension Tutorial](./getting_started/first-extension.md) - Step-by-step tutorial
- [Basic Concepts](./getting_started/basic-concepts.md) - Core concepts and terminology
- [Core Architecture](./architecture/core-architecture.md) - System design overview
- [Game Loop](./engine_integration/game-loop-lifecycle.md) - Execution flow

## **Architecture**
*Understanding the system design*
- [Core Architecture](./architecture/core-architecture.md) - Binary interface, function pointers, versioning
- [Memory Model](./architecture/memory-model.md) - Memory management, reference counting, lifecycle
- [Initialization Process](./architecture/initialization.md) - Startup sequence and interface loading

## **Type System**
*Variant, ClassDB, and type handling*
- [Variant Internals](./type_system/variant-internals.md) - Storage layout, type discrimination, conversions
- [ClassDB Internals](./type_system/classdb-internals.md) - Class registration and method binding
- [Template Specializations](./type_system/template-specializations.md) - Type traits, SFINAE, marshalling

## **Engine Integration**
*Interacting with Godot Engine*
- [Game Loop & Lifecycle](./engine_integration/game-loop-lifecycle.md) - Frame processing and callbacks
- [Object, Resource, and Node](./engine_integration/object-resource-node.md) - Core class hierarchies
- [Binding System](./engine_integration/binding-system.md) - Method, property, and signal bindings
- [Notification System](./engine_integration/notification-system.md) - Event propagation patterns
- [Singleton Interactions](./engine_integration/singleton-interactions.md) - Engine singleton usage
- [GDScript Interaction](./engine_integration/gdscript-interaction.md) - Cross-language communication
- [Class Registration & Inheritance](./engine_integration/class-registration-inheritance.md) - Registration types and behavior

## **Code Generation**
*How bindings are generated*
- [Binding Generator](./code_generation/binding-generator.md) - extension_api.json processing
- [Generated Patterns](./code_generation/generated-patterns.md) - Generated code structure

## **Build System**
*Building for different platforms*
- [Build Architecture](./build_system/build-architecture.md) - SCons configuration and toolchain
- [Platform Implementation](./build_system/platform-implementation.md) - Platform-specific builds

## **Advanced Topics**
*Complex patterns and optimization*
- [Complex Patterns](./advanced_topics/complex-patterns.md) - Advanced usage patterns
- [Error Handling & Debugging](./advanced_topics/error-debugging.md) - Debugging techniques
- [Performance Optimization](./advanced_topics/performance-optimization.md) - Optimization strategies
- [Thread Safety](./advanced_topics/thread-safety.md) - Concurrent programming patterns

## **API Reference**
*Technical reference documentation*
- [Error Macros](./api_reference/error-macros.md) - Error handling macro reference
- [Internal APIs](./api_reference/internal-apis.md) - Internal API documentation
- [Utility Functions](./api_reference/utility-functions.md) - Helper function reference

## **Examples**
*Complete working examples*
- [Simple Extension](./examples/simple-extension.md) - Basic extension demonstrating core concepts
- [Advanced Extension](./examples/advanced-extension.md) - Complex patterns and features
- [Integration Extension](./examples/integration-extension.md) - Third-party library integration

## **Guides**
*Task-specific tutorials*
- [Creating Custom Nodes](./guides/creating-custom-nodes.md) - Step-by-step node creation
- [Resource Management](./guides/resource-management.md) - Managing resources effectively
- [Signal Patterns](./guides/signal-patterns.md) - Signal usage best practices
- [Threading Safety](./guides/threading-safety.md) - Thread-safe extension development

## **Resources**
*Supporting materials*
- [Glossary](./resources/glossary.md) - Term definitions
- [FAQ](./resources/faq.md) - Frequently asked questions
- [Godot Documentation](https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/) - Godot Official GDExtension Documentation
- [Godot-CPP Library Source](https://github.com/godotengine/godot-cpp) - Godot CPP Github URL
- [Godot Engine Source](https://github.com/godotengine/godot) - Godot Engine Github URL
- [Community Discord](https://discord.gg/godotengine) - Godot Official Community Discord
- [Unofficial Community Discord](https://discord.gg/3NYTrm35) - Godot Cafe Unofficial Community Discord

## License
This documentation is provided under the same license as Godot: [MIT License](LICENSE)
