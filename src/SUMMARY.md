# The Xous Operating System

[Introduction](ch00-00-introduction.md)

## Getting started

- [Getting Started](ch01-00-getting-started.md)
    - [Hello, World!](ch01-02-hello-world.md)
    - [Hello, Renode!](ch01-03-hello-renode.md)

- [Server Architecture](ch02-00-server-architecture.md)
    - [Synchronization](ch02-04-synchronization.md)

- [Introducing the Kernel](ch03-00-introducing-the-kernel.md)
    - [Memory Layout](ch03-01-memory-layout.md)
    - [Hosted Mode](ch03-02-hosted-mode.md)
    - [Process Creation](ch03-03-process-creation.md)

- [Renode Emulation](ch04-00-renode-emulation.md)
    - [Platform Definition](ch04-01-platform-definition.md)
    - [Renode Startup Script](ch04-02-renode-startup-script.md)
    - [Python Extensions](ch04-03-python-extensions.md)
    - [Writing C# Peripherals](ch04-04-writing-cs-peripherals.md)

- [System Startup](ch05-00-system-startup.md)
    - [Arguments Structure](ch05-01-arguments.md)
    - [Xous Loader](ch05-02-loader.md)
    - [MiniELF Format](ch05-03-minielf.md)

- [Build System](ch06-00-build-system-overview.md)
    - [Testing Crates](ch06-01-testing-crates.md)
    - [Image Creation](ch06-02-create-image.md)

- [Messages](ch07-00-messages.md)
    - [Xous Names](ch07-01-xous-names.md) how to discover and connect with services in Xous
    - [Caller Idioms](ch07-02-caller-idioms.md) includes examples of non-synchronizing, synchronous, asynchronous, and deferred callback implementations.
       - [Non-synchronizing](ch07-03-nonsynchronizing.md)
       - [Synchronous](ch07-04-synchronizing.md)
       - [Asynchronous](ch07-05-asynchronous.md)
       - [Deferred Response](ch07-06-deferred.md)
       - [Forwarding](ch07-07-forwarding.md)

- [Graphics Toolkit](ch08-00-graphics.md)
    - [Modals](ch08-01-modals.md)
    - [Menus](ch08-02-menus.md)
