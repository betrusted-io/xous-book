# The Xous Operating System

[Introduction](ch00-00-introduction.md)

## Getting started

- [Getting Started](ch01-00-getting-started.md)
    - [Hello, World!](ch01-02-hello-world.md)
    - [Coding Style](ch01-03-coding-style.md)
    - [Xous by example](ch01-04-xous-by-example.md)
    - [Glossary & Jargon](ch01-05-glossary.md)

- [Server Architecture](ch02-00-server-architecture.md)
    - [Synchronization](ch02-04-synchronization.md)

- [Introducing the Kernel](ch03-00-introducing-the-kernel.md)
    - [Memory Management in Xous](ch03-01-memory-layout.md)
    - [Hosted Mode](ch03-02-hosted-mode.md)
    - [Process Creation](ch03-03-process-creation.md)
    - [Debugging Programs with GDB](ch03-04-debugging-programs.md)

- [Renode Emulation](ch04-00-renode-emulation.md)
    - [Writing C# Peripherals](ch04-04-writing-cs-peripherals.md)

- [Xous Operating System Startup](ch05-00-system-startup.md)
    - [System Arguments](ch05-01-arguments.md)
    - [Xous Loader](ch05-02-loader.md)
    - [MiniELF File Format](ch05-03-minielf.md)

- [Xous Build System Overview](ch06-00-build-system-overview.md)
    - [Testing Crates](ch06-01-testing-crates.md)
    - [Xous Image Creation](ch06-02-create-image.md)
    - [Target Specification and Hardware Registers with UTRA](ch06-03-target-specification.md) read here to learn how hardware registers are handled.

- [Messages and Message Passing](ch07-00-messages.md)
    - [Xous Names](ch07-01-xous-names.md) how to discover and connect with services in Xous
    - [Caller Idioms](ch07-02-caller-idioms.md) includes examples of non-synchronizing, synchronous, asynchronous, and deferred callback implementations.
       - [Non-synchronizing](ch07-03-nonsynchronizing.md)
       - [Synchronous](ch07-04-synchronizing.md)
       - [Asynchronous](ch07-05-asynchronous.md)
       - [Deferred Response](ch07-06-deferred.md)
       - [Forwarding](ch07-07-forwarding.md)
   -  [Messaging Performance](ch07-08-performance.md)

- [Graphics Toolkit](ch08-00-graphics.md)
    - [Modals](ch08-01-modals.md)
    - [Menus](ch08-02-menus.md)

- [The Plausibly Deniable DataBase (PDDB) Overview](ch09-00-pddb-overview.md)
  - [Basis Internal Structure](ch09-01-basis.md)
  - [Deriving The PDDB's Keys](ch09-02-rootkeys.md)
  - [Native API](ch09-03-api-native.md)
  - [`std` API](ch09-04-api-std.md)
  - [Testing and CI](ch09-05-testing.md)
  - [Backups](ch09-06-backups.md)
  - [Security and Deniability](ch09-07-discussion.md)

- [Encrypted Swap](ch10-00-swap-overview.md)
