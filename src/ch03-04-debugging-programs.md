# Debugging with GDB

The kernel supports enabling the `gdb-stub` feature which will provide a gdb-compatible server on the 3rd serial port. This server can be used to debug processes that are running, and when a process crashes the kernel will automatically halt the process for debugging.

When using the debugger, many features are supported:

* Listing processes on the system
* Attaching to a given process
* Listing threads
* Examining and updating memory
* Examining and updating registers
* Single-stepping (non-XIP processes)
* Inserting breakpoints (non-XIP processes)

The following features are NOT SUPPORTED:

* Watchpoints
* Inserting breakpoints in XIP processes
* Single-stepping XIP processes

## Enabling the debugger

You probably want to set `debug = true` inside `Cargo.toml`. This will add debug symbols to the resulting ELF binaries which greatly enhance the debugging experience. Otherwise the debugger will have to guess where 

To enable the debugger, pass `--gdb-stub` to `xtask`. For example:

```text
cargo xtask app-image-xip --gdb-stub mtxchat --feature efuse --feature tls
```

Then, flash the resulting image to the target device as normal.

## Attaching to the debugger

If you're using Renode, then you can connect gdb to `localhost:3456`:

```text
riscv-none-elf-gdb -ex 'tar ext :3456'
```

On real hardware, you will first need to re-mux the serial port so that gdb is visible on serial. Then you can connect gdb to the target serial port.

## Debugging a process

Within the gdb server, you can switch which file you're debugging. For example, to debug the ticktimer, run:

```text
(gdb) file target/riscv32imac-unknown-xous-elf/release/xous-ticktimer
```

After setting the ELF file you will need to attach to the process. Use `mon pr` or `monitor process` to list available processes. Then, use `attach` to attach to a process:

```text
(gdb) mon pr
Available processes:
   1   kernel
   2   xous-ticktimer
   3   xous-log
   4   xous-names
   5   xous-susres
   6   libstd-test
(gdb) att 2
Attaching to process 2
[New Thread 2.2]
[New Thread 2.3]
0xff802000 in ?? ()
(gdb)
```

You can switch processes by sending `att` to a different PID.

## Debugging a thread

To list threads, use `info thr`:

```text
(gdb) info thr
  Id   Target Id         Frame
* 1    Thread 2.255      0xff802000 in ?? ()
  2    Thread 2.2        xous::definitions::Result::from_args (src=...) at src/definitions.rs:474
  3    Thread 2.3        xous::definitions::Result::from_args (src=...) at src/definitions.rs:474
(gdb)
```

To switch threads, use `thr [n]`:

```text
(gdb) thr 2
[Switching to thread 2 (Thread 2.2)]
#0  xous::definitions::Result::from_args (src=...) at src/definitions.rs:474
474             match src[0] {
(gdb)
```

**Important Note**: GDB thread numbers are different from Xous thread numbers! GDB Always starts with `1`, but Xous may have any number of threads running. GDB pays attention to the first column, and you are most likely interested in the second column.
