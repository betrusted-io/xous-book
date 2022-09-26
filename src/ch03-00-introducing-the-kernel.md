# Introducing the Kernel

Xous is a microkernel design that tries to keep as little as possible inside the main kernel. Instead, programs can start "Servers" that process can connect to in order to accomplish a task.

Processes are isolated, and therefore an MMU is strongly recommended. One process can have multiple threads, and processes cannot interact with one another except by passing Messages.

A Message is a piece of data that can be sent to a Server. Messages contain one `usize` ID field that may be used to identify an opcode, and may additionally contain some memory or some `usize` scalars.

Additionally, a Message may be either blocking, in which case it will wait for the Server to respond, or non-blocking, where they will return immediately.

"Drivers" are really just Servers. For example, to print a string to the console, send a `StandardOutput` (1) opcode to the server "xous-log-server " with a `&[u8]` attached to some memory. The process will block until the server is finished printing.

The entire Xous operating system is built from these small servers, making it easy to work on one component at a time.

## Memory and mapping

Memory is obtained by issuing a `MapMemory` syscall. This call can optionally provide a physical address to map. If no memory is specified, a random physical page is provided. The process has no way of knowing the physical address of the page.

If the caller allocates memory from the primary region, it will be zeroed. If it allocates memory from an ancillary region such as a registor or a framebuffer, then that memory will not be initialized.

Processes can use this to allocate memory-mapped regions in order to create drivers.

## Interrupts

Processes can allocate interrupts by calling the `ClaimInterrupt` call. If an interrupt has not been used, then that process will become the new owner of that interrupt. This syscall requires you to specify an address of a function to call, and you may optionally provide an argument to pass to the function handler.

There is no way to disable interrupts normally, except by handling an interrupt. That is, interrupts are disabled inside of your interrupt handler, and will be re-enabled after your interrupt handler returns.

There are a very limited set of functions that may be called during an interrupt handler. You may send nonblocking Messages and allocate memory, for example. However you may not Yield or send blocking Messages.

A common pattern for "disabling interrupts" is to come up with an interrupt that does nothing but trigger on-demand and handling the requisite code in that function. This is used by the suspend/resume server, for example, in order to ensure nothing else is running when the system is powering down.

## Supported Platforms: RISC-V 32 and Hosted

Xous currently supports two platforms: RISC-V 32 and Hosted mode.

RISC-V 32 is the hardware that ships in Betrusted and Precursor, and is what is available in the Renode emulator.

An additional platform is `Hosted` mode, which targets your desktop machine. This can be used to debug builds using desktop-class debuggers such as `rr` or even just `gdb`. You can also use profilers in order to discover where code performance can be improved.

`Hosted` mode is discussed in [more detail later](ch03-02-hosted-mode.md)
