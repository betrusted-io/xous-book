# Process Creation

Creating processes is a fundamental requirement of modern operating systems above a certain size. Xous supports process creation, although it does not prescribe an executable format nor does it even have a built-in loader.

Process creation arguments vary depending on the platform being targeted, making this one of the less portable aspects of Xous. All platforms support the `CreateProcess` syscall, however the arguments to this syscall vary widely.

## Creating Processes in Hosted Mode

In Hosted mode, the `ProcessArgs` struct contains a full command line to be passed directly to the shell. This is actually used by the kernel during its init routine when it spawns each child process of PID 1.

Internally, the parent process is responsible for launching the process as part of the `create_process_post()` that gets called after the successful return of the `CreateProcess` syscall. As part of this, the hook sets various environment variables for the child process such as its 16-byte key stored in the `XOUS_PROCESS_KEY` variable, as well as the PID stored in the `XOUS_PID` variable.

## Creating Processes in Test Mode

Test mode is a special case. Tests don't want to depend on files in the filesystem, particularly as multiple tests are running at the same time. To work around this, processes are created as threads. This is a special case intended to support heavily-parallel machines that can run all thread tests simultaneously, and is not normally used.

## Creating Processes on Native Hardware (e.g. RISC-V)

Process creation on real hardware requires a minimum of six pieces of information. These are all defined in the `ProcessInit` struct, which gets passed directly to the kernel:

```rust,noplayground,ignore
pub struct ProcessInit {
    // 0,1 -- Stack Base, Stack Size
    pub stack: crate::MemoryRange,
    // 2,3 -- Text Start, Text Size
    pub text: crate::MemoryRange,
    // 4 -- Text destination address
    pub text_destination: crate::MemoryAddress,
    // 5 -- Entrypoint (must be within .text)
    pub start: crate::MemoryAddress,
}
```

The `stack` defaults to 128 kB growing downwards from `0x8000_0000`.

`text` refers to a region of memory **INSIDE YOUR PROGRAM** that will be detached and moved to the child process. This memory will form the initialization routine for the child process, and should contain no `.bss` or `.data` sections, unless it also contains code to allocate and set up those sections.

`text_destination` describes the offset where `text` will be copied. This address is determined by the link address of your initialization program. 

The `start` is the address where the program counter will start. This is the address of your program's entrypoint. It must reside within the allocated text section, beginning at `text_destination`.

### Native Hardware Entrypoint

The entrypoint for native hardware takes four arguments. When combined, these four arguments form a Server ID that can be used for sending additional data from the parent process to the child. An example loader program might look like the following:

```rust,noplayground,ignore
pub extern "C" fn init(a1: u32, a2: u32, a3: u32, a4: u32) -> ! {
    let server = xous::SID::from_u32(a1, a2, a3, a4);
    while let Ok(xous::Result::Message(envelope)) =
        xous::rsyscall(xous::SysCall::ReceiveMessage(server))
    {
        match envelope.id().into() {
            StartupCommand::WriteMemory => write_memory(envelope.body.memory_message()),
            StartupCommand::FinishStartup => finish_startup(server, envelope),
            StartupCommand::PingResponse => ping_response(envelope),

            _ => panic!("unsupported opcode"),
        }
    }
    panic!("parent exited");
}
```

This compiles down to a very efficient program that can be used to load a larger program into the new address space. Memory is written using the `WriteMemory` opcode to load new pages into the nacent process, and `FinishStartup` is used to shut down the server and jump to the new process entrypoint.

### Limitations of Created Processes

**NOTE:** The following is subject to fixes in the kernel, and do not currently apply. This information is presented here in order to explain oddities observed when these features are implemented.

Newly-created processes cannot create servers with a predefined Server ID. They can only create randomized servers.

Processes created using `CreateProcess` are not ever scheduled to run. Parent processes must donate their quantum to child processes in order for them to run. This is done with a special syscall.

When a parent process exits, all child processes will also exit. This is because those processes will not be scheduled anymore, so there's no point in letting them continue to run.