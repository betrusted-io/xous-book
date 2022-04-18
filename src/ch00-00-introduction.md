# Introduction

This book is a work in progress. Many chapters are placeholders and will appear blank.

The book is written for two audiences: kernel maintainers, and application developers.

Chapters [2 (Server Architecture)](ch02-00-server-architecture.md), [3 (Introducing the Kernel)](ch03-00-introducing-the-kernel.md), and [5 (System Startup)](ch05-00-system-startup.md) are primarily for kernel maintainers and system programmers.

Chapters [1 (Getting Started)](ch01-00-getting-started.md), [4 (Renode Emulation)](ch04-00-renode-emulation.md), [6 (Build System Overview)](ch06-00-build-system-overview.md), [7 (Messages)](ch07-00-messages.md) and [8 (Graphics)](ch08-00-graphics.md) are more appropriate for application developers.

----------------

**Xous** is a collection of small, single purpose **Servers** which respond to **Messages**. The xous **Kernel** delivers Messages to Servers, allocates processing time to Servers, and transfers memory ownership from one Server to another. Every xous Server contains a central loop that receives a Message, matches the Message **Opcode**, and runs the corresponding rust code. When the operation is completed, the Server waits to receive the next Message at the top of the loop, and processing capacity is released to other Servers. Every service available in xous is implemented as a Server. Every user application in xous is implemented as a Server.

There are only three "well known" Servers which are always available to receive Messages, and run the requested Opcode:
- The `xous-name-server` maintains a list of all registered Servers by name, and guards a randomised 16bit **Server ID** for each of the Servers. The xous-name-server arbitrates the flow of Messages between Servers.
- The `ticktime-server` provides time related services.
- The `xous-log-server` provides logging services.

The remaining servers are not "well known" - meaning that the `xous-name-server` must be consulted to obtain a Connection ID in order to send a Message to the Server. Such Servers include `aes` `com` `dns` `gam` `jtag` `keyboard` `llio` `modals` `net` `pddb` `trng`.

Every **Message** contains a **Connection ID** and an **Opcode**. The Connection ID is a "delivery address" for the recipient Server, and the Opcode specifies a particular operation provided by the recipient Server. There are two flavours of messages in xous:
- **Scalar messages** are very simple and very fast. Scalar messages can transmit only 4 u32 sized arguments.
- **Memory messages** can contain larger structures, but they are slower. A struct sent in a Memory Message must implement `#[derive(rkyv::Archive, rkyv::Serialize, rkyv::Deserialize)]` so that it can be serialized into a buffer by the sender and deserialized by the recipient
