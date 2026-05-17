# Glossary & Jargon

`bao1x` - The [Baochip-1x SoC](https://github.com/baochip/baochip-1x).

`baosec` - Internal development name of a fully-featured hardware password manager based on the `bao1x` SoC.

`betrusted-soc` - The name of the [FPGA-based SoC](https://github.com/betrusted-io/betrusted-soc) that powers `Precursor`

`BDMA` - BIO DMA (Direct Memory Access). DMA mediated by a BIO CPU.

`BIO` - Bao I/O. A quad-core Risc-V, register-mapped I/O processor with register-mapped inter-processor communication FIFOs.

`blocking` - A message type that stalls program execution until it returns.

`boot0` - On `bao1x`, a small, indelible code stub that sets up initial security and checks the validity of `boot1`.

`boot1` - On `bao1x`, an updateable machine-mode program that mediates firmware updates, audits and security.

`CID` - A `connection` ID. In order to keep `SID`s secret, the `CID` is effectively a lookup table to `SID`s that is maintained by the kernel.

`dabao` - A breakout board for the `bao1x` targeted at developers and system integrators.

`Drop` - A trait in Rust that is called when an object goes out of scope. Frequently used in Xous to automatically respond to blocking message types.

`client` - Any program attempting to talk to a `server`. Servers can be clients of other servers. Even the kernel can be a client of another server.

`connection` - A routable path between a client and a `server`. There can be up to 32 connections per `PID`.

`HAL` - Hardware abstraction layer.

`hosted mode` - An emulation mode for Xous where the kernel runs on your local PC

`keystore` - A userspace service that manages security keys for the `bao1x` platform. Must be at `PID` 3 when present. In Sv32 mode, the underlying hardware is aware of the current `PID` and will deny access to data slots if the wrong `PID` is running. Note that bootloaders have free reign of the system.

`lend` - In the context of `memory message`s, a page of memory that has been mapped into a `server`'s address space and marked as read-only. The page is returned to the caller when the message is `Drop`ped by the server.

`loader` - A program that runs in machine mode that copies the initial process set and sets up virtual memory for the kernel.

`memory message` - A message that uses virtual memory pages to communicate. A full `page` of memory is mapped into the target `server`'s process space. The `page` can be `send`, `lend`, or `mutable lend` type.

`message` - An abstract type that is used to communicate between clients and `server`s. Messages can be `scalar` or `memory`, `blocking` and `nonblocking`.

`mutable lend` - In the context of `memory message`s, a page of memory that has been mapped into a `server`'s address space and marked as read/write. The page (and any updated data within) is returned to the caller when the message is `Drop`ped by the server.

`nonblocking` - A message type that continues execution without waiting for the receiver to acknowledge receipt. Basically, async communications. `Scalar` messages are assumed to be non-blocking unless specified as blocking.

`page` - The smallest mappable unit of memory in Sv32. In Xous, this is set to 4096 bytes.

`Precursor` - A fully-featured system for password management built around the `betrusted-soc` FPGA design. See [precursor.dev](https://precursor.dev).

`scalar message` - A message whose arguments fit entirely within the available registers of the CPU (maximum of 5, 32-bit arguments). Scalar messages are distinguished from memory messages in that they are much faster to send and return.

`send` - In the context of `memory message`s, a page of memory that has been mapped into `server`'s address space with no expectation of return. The page now belongs to the target server forever, until it `Drop`s the page.

`thread` - A scheduled unit of computation. Each PID can have up to 30 threads. Note that thread 0 is always present and it is the interrupt handler thread.

`TID` - The ID number of a thread. `TID` 0 is the interrupt handler thread.

`PID` - The process ID. This defines an isolated virtual memory name space. There can be up to 255 PIDs in Xous, where PID 1 is always the kernel (PID 0 is not valid). As a rule, each `service` is located in its own PID, but for tightly coupled `service`s more than one can be packed into a PID.

`server` - A routable messaging endpoint, specified with a 128-bit server ID. The server ID may be randomly generated, in which case, it is considered secret and not directly discoverable; it has to be looked up and approved via `xous-names`.

`service` - A collection of `server`s that provide a service. Generally confined to a single PID. As a microkernel, many functions traditionally provided by a monolithic kernel are actually provided by userspace services. Examples: `ticktimer`, `xous-names`.

`Sv32` - The RISC-V standard for virtual memory using 32-bit addresess and 2-level page tables.

`target` (Rust) - A compiler specification in the form of `<architecture>-<vendor>-<os>-<abi>`. Example: `riscv32imac-unknown-xous-elf`

`target` (Xous) - A specific combination of board & application software, yielding a binary image that can be programmed onto the board. Examples: `baosec`, `dabao`, `app-image`.

`ticktimer` - The equivalent of a scheduler in monolithic kernels. `ticktimer` implements sleeps, mutex's, and elapsed time sense for the overall system.

`TOFU` - Trust On First Use. A method for limiting access where the first contact is considered to be the only trusted contact: requires a strong assumption that the boot environment is safe.

`xous-log` - A service that provides console logging and interaction.

`xous-names` - A lookup service that can translate free-form utf-8 strings that name a service into `CID`s. Internally, `xous-names` maintains a lookup table between the server name and its `SID`, along with a count of connections to the server. It can enforce a maximum connection limit, enabling a `TOFU`-style access control between trusted and untrusted services.

`xous-swapper` - A userspace service that manages swap memory on behalf of the kernel. Must always be at `PID` 2 when present.