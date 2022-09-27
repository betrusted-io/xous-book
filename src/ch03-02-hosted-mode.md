# Hosted Mode

Hosted mode may be built by running `cargo xtask run`. This causes Xous to be compiled using your native architecture rather than building for `riscv32imac-unknown-xous-elf`. Your native architecture is probably 64-bits, and has a lot more memory than Betrusted does. Xous also runs in userspace, which means a lot of things end up being very different in this mode.

The API is designed to abstract away these differences so that programs may run seamlessly on both Hosted and Native (RISC-V 32) mode.

## The Kernel as a Process

When you build processes using `cargo xtask run`, the kernel is compiled as an ordinary, native program. This program can be run by simply running `./target/release/kernel`. If you run this by itself after running `cargo xtask run`, you'll see the following output:

```sh
$ ./target/release/kernel
KERNEL: Xous server listening on 127.0.0.1:1238
KERNEL: Starting initial processes:
  PID  |  Command
-------+------------------
```

The kernel simply acts as a router, passing messages between processes. This poses some challenges because processes need to be able to connect to one another, and the kernel needs to be able to match a network connection to a given process. Additionally, there needs to be a list of initial processes to start.

## Initial Processes

In order to get a list of initial processes, they are simply all passed on the command line. For example, we can run the kernel with a log server and see the following output:

```sh
$ ./target/release/kernel ./target/release/log-server
KERNEL: Xous server listening on 127.0.0.1:21183
KERNEL: Starting initial processes:
  PID  |  Command
-------+------------------
   2   |  ./target/release/log-server
LOG: my PID is 2
LOG: Creating the reader thread
LOG: Running the output
LOG: Xous Logging Server starting up...
LOG: Server listening on address SID([1937076088, 1735355437, 1919251245, 544367990])
LOG: my PID is 2
LOG: Counter tick: 0
```

From this output, we can see that the kernel has started the log server for us. Multiple initial processes may be specified:

```sh
$ ./target/release/kernel ./target/release/log-server ./target/release/xous-names
KERNEL: Xous server listening on 127.0.0.1:3561
KERNEL: Starting initial processes:
  PID  |  Command
-------+------------------
   2   |  ./target/release/log-server
   3   |  ./target/release/xous-names
LOG: my PID is 2
LOG: Creating the reader thread
LOG: Running the output
LOG: Xous Logging Server starting up...
LOG: Server listening on address SID([1937076088, 1735355437, 1919251245, 544367990])
LOG: my PID is 2
LOG: Counter tick: 0
INFO:xous_names: my PID is 3 (services/xous-names/src/main.rs:360)
INFO:xous_names: started (services/xous-names/src/main.rs:375)
```

## Launching a Process

Processes are launched in the kernel by setting a series of environment variables and then spawning a new process. The following environment variables are currently used:

| Variable          | Description                                                       |
| ----------------- | ----------------------------------------------------------------- |
| XOUS_SERVER       | The IP and TCP port of the kernel                                 |
| XOUS_PID          | The unique process ID of this kernel, assigned by the Xous kernel |
| XOUS_PROCESS_NAME | The process name, currently taken from the executable name        |
| XOUS_PROCESS_KEY  | An 8-byte hex-encoded key that uniquely identifies this process   |

A thread is created for this process to handle it and to route messages within the kernel. The `XOUS_PROCESS_KEY` is effectively a single-use token that is unique per process and is used to match a process within the kernel.

When the process launches it should establish a connection to the kernel by connecting to `XOUS_SERVER` and sending `XOUS_PROCESS_KEY`. This will authenticate the process with the kernel ane enable it to send and receive messages.

The initial handshake has the following layout:

| Offset (Bytes) | Size | Meaning                          |
| -------------- | ---- | -------------------------------- |
| 0              | 1    | Process ID of connecting process |
| 1              | 8    | 8-byte process key               |

## Sending and Receiving Syscalls

In Hosted mode, syscalls are sent via a network connection. Because pointers are unsafe to send, `usize` is defined on Hosted mode as being 32-bits. Additionally, most syscalls will return `NotImplemented`, for example it does not make sense to create syscalls such as `MapMemory`.

Messages function normally in Hosted mode, however they are more expensive than on real hardware. Because messages get sent via the network, the entire contents of a Memory message must be sent across the wire.

Eight 32-bit values are sent, and these may be followed by any data in case there is a Memory message.

| Offset (Bytes) | Usage (Calling)                           |
| -------------- | ----------------------------------------- |
| 0              | Source thread ID                          |
| 4              | Syscall Number                            |
| 8              | Arg 1                                     |
| 12             | Arg 2                                     |
| 16             | Arg 3                                     |
| 20             | Arg 4                                     |
| 24             | Arg 5                                     |
| 28             | Arg 6                                     |
| 32             | Arg 7                                     |
| 36             | Contents of any buffer pointed to by args |

The process should expect a return, and should block until it gets a response. When it gets a response, a memory buffer may be required that is the same size as the buffer that was sent. The contents of this buffer will be appended to the network packet in the same manner as the calling buffer. If the message is a Borrow, then this data will be the same as the data that was sent. If it is a MutableBorrow, then the server may manipulate this data before it returns.

| Offset (Bytes) | Usage (Return)                  |
| -------------- | ------------------------------- |
| 0              | Target thread ID                |
| 4              | Return type tag                 |
| 8              | Arg 1                           |
| 12             | Arg 2                           |
| 16             | Arg 3                           |
| 20             | Arg 4                           |
| 24             | Arg 5                           |
| 28             | Arg 6                           |
| 32             | Arg 7                           |
| 36             | Contents of any returned buffer |


## Threading

All Xous syscalls go to the kernel, however certain syscalls are simply stubs. One example of this is threading, where the kernel has no way of actually launching a thread.

The application is responsible for creating new threads, and may do so either by "sending" a `CreateThread` call to the kernel or by creating a native thread using `std::Thread::spawn()`.

When launching a thread with `CreateThread`, the kernel will allocate a new "Xous TID" and return that to the application. The application will then launch its new thread and set the local `THREAD_ID` variable to this ID. This ID will be used as part of the header when sending syscalls to the kernel, and will be used to delegate responses to their waiting threads.

If an application calls `std::Thread::spawn()` then it will not have a `THREAD_ID` set. When the thread attempts to send a syscall, hosted mode will notice that `THREAD_ID` is None. When this occurs, Hosted mode will create a "fake" thread ID (starting at TID 65536) and call `SysCall::CreateThread(ThreadInit {})` to register this new ID. Then all subsequent calls will use this fake thread ID.
