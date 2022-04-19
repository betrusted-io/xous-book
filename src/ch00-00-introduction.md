# Introduction

This book is a work in progress. Many chapters are placeholders and will appear blank.

The book is written for two audiences: kernel maintainers, and application developers.

Chapters [2 (Server Architecture)](ch02-00-server-architecture.md), [3 (Introducing the Kernel)](ch03-00-introducing-the-kernel.md), and [5 (System Startup)](ch05-00-system-startup.md) are primarily for kernel maintainers and system programmers.

Chapters [1 (Getting Started)](ch01-00-getting-started.md), [4 (Renode Emulation)](ch04-00-renode-emulation.md), [6 (Build System Overview)](ch06-00-build-system-overview.md), [7 (Messages)](ch07-00-messages.md) and [8 (Graphics)](ch08-00-graphics.md) are more appropriate for application developers.

----------------

## Architecture

**Xous** is a collection of small, single purpose **Servers** which respond to **Messages**. The Xous **Kernel** delivers Messages to Servers, allocates processing time to Servers, and transfers memory ownership from one Server to another. Every xous Server contains a central loop that receives a Message, matches the Message **Opcode**, and runs the corresponding rust code. When the operation is completed, the Server waits to receive the next Message at the top of the loop, and processing capacity is released to other Servers. Every service available in xous is implemented as a Server. Every user application in xous is implemented as a Server.

Architecturally, Xous is most similar to [QNX](https://www.qnx.com/developers/docs/6.4.1/neutrino/getting_started/s1_msg.html), another microkernel message-passing OS.

### Servers

There are only a few "well known" Servers which are always available to receive Messages, and run the requested Opcode:
- The `xous-name-server` maintains a list of all registered Servers by name, and guards a randomised 128-bit **Server ID** for each of the Servers. The xous-name-server arbitrates the flow of Messages between Servers.
- The `ticktimer-server` provides time and time-out related services.
- The `xous-log-server ` provides logging services.
- The `timeserverpublic` provides real-time (wall-clock time) services. It is only accessed via `std::time` bindings.

The remaining servers are not "well known" - meaning that the `xous-name-server` must be consulted to obtain a Connection ID in order to send the Server a Message. Such Servers include `aes` `com` `dns` `gam` `jtag` `keyboard` `llio` `modals` `net` `pddb` `trng`.

### Messages, aka IPC

Every **Message** contains a **Connection ID** and an **Opcode**. The Connection ID is a "delivery address" for the recipient Server, and the Opcode specifies a particular operation provided by the recipient Server. There are two flavours of messages in xous:

- **Scalar messages** are very simple and very fast. Scalar messages can transmit only 4 u32 sized arguments.
- **Memory messages** can contain larger structures, but they are slower. They "transmit" page-sized (4096-byte) memory chunks.

Rust `struct`s need to be serialized into bytes before they can be passed using Memory Messages. Xous provides convenience bindings for `rkyv`, so any `struct` fully-annotated with `#[derive(rkyv::Archive, rkyv::Serialize, rkyv::Deserialize)]` can be serialized into a buffer by the sender and deserialized by the recipient.

The most simple Server communication involves a **non-synchronizing** "fire and forget" style of Messaging. The Sender sends a Message and continues processing immediately. The Recipient will receive the Message when it arrives, and process the Opcode accordingly. End of story. The ownership of the Message memory passes from the Sender to the Recipient and is Dropped by the Recipient. While there will be a delay before the Message is received - the sequence is assured. In the code, these are referred to as either `Scalar` Scalar Messages or `Send` Memory Messages.

Alternatively, A Server can send a **synchronous** Message, and wait (block) until the Recipient completes the operation and responds. In this arrangement, the Message memory is merely lent to the Recipient (read-only or read-write) and returned to the Sender on completion. While the sender Server "blocks", its processing quanta is not wasted, but also "lent" to the Recipient Server to complete the request promptly. In the code, these are referred to as either `BlockingScalar` Scalar Messages, or `Borrow` or `MutableBorrow` Memory Messages. `Borrow` messages are read-only, `MutableBorrow` are read-write, with semantics enforced by the Rust borrow checker.

**asynchronous** Message flow is also possible. The Sender will send a non-synchronous Message and include its own Connection ID as a "return address". The Recipient Server will complete the operation, and then send a non-synchronous Message in reply to the Connection ID of the sender.

A Server may also send a synchronous Message and wait for a **deferred-response**. This setup is needed when the recipient Server cannot formulate a reply within a single pass of the event loop. Rather, the recipient Server must "park" the request and continue to process subsequent Messages until the original request can be satisfied. The request is "parked" by either saving the `msg.sender` field (for Scalar messages) or keeping a reference to the `MessageEnvelope` (for Memory messages). Memory Messages automatically return-on-Drop, relying on the Rust borrow checker and reference counting system to enforce implicit return semantics.
