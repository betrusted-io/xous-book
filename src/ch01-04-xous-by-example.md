# Xous by Example

This chapter is for developers who like to see an example of how things work.

The code here is a snapshot of the `baosec` target development branch, including the `vault2` application. It was built with the following command line:

`cargo xtask baosec --no-timestamp --feature usb --feature with-pddb --feature aestests --kernel-feature debug-proc`

- `--no-timestamp` suppresses timestamp generation in the kernel, allowing for reproducible builds
- `--feature usb` specifies that we want the USB stack features to be turned on in all services
- `--feature with-pddb` specifies that we're using the PDDB in all our programs. Eventually this flag will be deprecated in this target as it will be assumed by default
- `--feature aestests` was added because at the moment the documentation was written, some AES testing was also being done
- `--kernel-feature debug-proc` is what allows me to generate the tables you see below. Kernel and loader features are separate from userspace application features, and thus those features have to be specified with `--kernel-feature` or `--loader-feature`, respectively.

The above command generated three binary artifacts, `loader.uf2`, `swap.uf2`, and `xous.uf2`. These were all copied onto the `baosec` board using the USB loader interface.

## Processes

Here's what the process table looks like in this target image:

```text
Process 1 state: Ready(100)  TID: 2  Memory mapping: (satp: 0x80461179, mode: 1, ASID: 1, PPN: 61179000) conns:0/32 kernel
Process 2 state: Running(0)  TID: 2  Memory mapping: (satp: 0x808611bf, mode: 1, ASID: 2, PPN: 611bf000) conns:3/32 xous-swapper
Process 3 state: Sleeping  TID: 2  Memory mapping: (satp: 0x80c611b8, mode: 1, ASID: 3, PPN: 611b8000) conns:5/32 keystore
Process 4 state: Sleeping  TID: 2  Memory mapping: (satp: 0x810611b1, mode: 1, ASID: 4, PPN: 611b1000) conns:2/32 xous-ticktimer
Process 5 state: Ready(1000)  TID: 3  Memory mapping: (satp: 0x814611aa, mode: 1, ASID: 5, PPN: 611aa000) conns:2/32 xous-log
Process 6 state: Ready(100)  TID: 2  Memory mapping: (satp: 0x818611a3, mode: 1, ASID: 6, PPN: 611a3000) conns:2/32 xous-names
Process 7 state: Sleeping  TID: 2  Memory mapping: (satp: 0x81c6119c, mode: 1, ASID: 7, PPN: 6119c000) conns:7/32 usb-bao1x
Process 8 state: Sleeping  TID: 3  Memory mapping: (satp: 0x82061195, mode: 1, ASID: 8, PPN: 61195000) conns:12/32 bao1x-hal-service
Process 9 state: Sleeping  TID: 2  Memory mapping: (satp: 0x8246118e, mode: 1, ASID: 9, PPN: 6118e000) conns:5/32 modals
Process 10 state: Sleeping  TID: 5  Memory mapping: (satp: 0x82861187, mode: 1, ASID: 10, PPN: 61187000) conns:8/32 pddb
Process 11 state: Sleeping  TID: 3  Memory mapping: (satp: 0x82c61180, mode: 1, ASID: 11, PPN: 61180000) conns:9/32 bao-video
Process 12 state: Sleeping  TID: 3  Memory mapping: (satp: 0x83061172, mode: 1, ASID: 12, PPN: 61172000) conns:10/32 bao-console
Process 13 state: Sleeping  TID: 2  Memory mapping: (satp: 0x8346116a, mode: 1, ASID: 13, PPN: 6116a000) conns:18/32 vault2
```

Note how each process has an ID number (1-13) in this case. This corresponds to an `ASID` in `Sv32`. The kernel is `PID` 1.

Each process has a state (`Ready`, `Running`, `Sleeping`), along with the currently scheduled `TID` or thread-ID.
Furthermore, each process has up to 32 connections that it can make to other servers.

Note that PID 1-11 are located in `xous.uf2`, whereas PID 12 and 13 are located in `swap.uf2`. There's no way to distinguish this from the kernel's perspective once the system is up and running, but it's important to note because copying just one of the two images can lead to ABI incompatibilities between the applications in `swap.uf2` and the core services in `xous.uf2`.

## Servers

Here's the list of servers:

```text
 idx | pid | process              | sid
 --- + --- + -------------------- | ------------------
   0 |   5 | xous-log             | SID([73756f78, 676f6c2d, 7265732d, 20726576])
   1 |   6 | xous-names           | SID([73756f78, 6d616e2d, 65732d65, 72657672])
   2 |   7 | usb-bao1x            | SID([4c403d60, 972d79bb, febdf19d, a4d054cf])
   3 |  10 | pddb                 | SID([8e004f62, 521e147d, bcb60ce, a6564b06])
   4 |   8 | bao1x-hal-service    | SID([1c711b06, 56546446, c7dd75de, 2952465e])
   5 |   9 | modals               | SID([2597c356, c6c3a4df, 1bde0f1, 42437b9a])
   6 |  11 | bao-video            | SID([724e4018, 83d15c9e, a1954161, 499574f6])
   7 |   8 | bao1x-hal-service    | SID([6279656b, 6472616f, 756f625f, 7265636e])
   8 |   2 | xous-swapper         | SID([982b7882, eb5d20a2, 53f79ccb, 44d37d10])
   9 |   3 | keystore             | SID([1df38619, 790f559b, e1d4c54d, bec5c5e9])
  10 |   4 | xous-ticktimer       | SID([6b636974, 656d6974, 65732d72, 72657672])
  11 |  12 | bao-console          | SID([fa38d1ef, 9fbe4c20, 4d1223ea, 216750de])
  12 |  13 | vault2               | SID([db457098, edc561f1, f7c07970, 9d3a5509])
  13 |   8 | bao1x-hal-service    | SID([b75041bd, 1c527620, 2b7b709d, 6fd0c190])
  14 |   8 | bao1x-hal-service    | SID([656d6974, 76726573, 75707265, 63696c62])
  15 |   8 | bao1x-hal-service    | SID([f97d0308, 5804dfc7, f2a324a5, c61af8a1])
  16 |  12 | bao-console          | SID([1b58af85, 703ceb4, 459ff738, 29425226])
  17 |   7 | usb-bao1x            | SID([5b658ac3, e2b2f065, dfc85a, 49b626cd])
  18 |  10 | pddb                 | SID([9fe99d87, 686aaa29, dd63834a, f31ab336])
  19 |  11 | bao-video            | SID([2b65be06, c6774932, f19a8610, 92cba265])
  20 |  13 | vault2               | SID([25b43d0f, 7c9e09d5, a11ada33, 675a43a2])
  21 |  13 | vault2               | SID([cb57f1d5, 424ebe49, fd835279, cea1190b])
  22 |  13 | vault2               | SID([bdb3ed67, 9cad154, 7940ec72, 2eaae5a4])
  23 |  13 | vault2               | SID([519f2270, fd0bafb9, b270c3e3, a07d138d])
  24 |  13 | vault2               | SID([b7de855c, 8ed779f7, 29ff9d7, e6514744])
  25 |  13 | vault2               | SID([7c38c99e, bfc80fd4, 1e9bf375, a377920])
  26 |  13 | vault2               | SID([3cebb08d, 17979315, 1ebd1cf8, f3569a0])
```

Here you can see the list of servers allocated in the system. The `SID` is a 128-bit number. It can be either a
cryptographically random number provided by the kernel, or a well-known number specified by the server itself.
The choice is up to the server: some services want to be connectable by everyone, and so they use a fixed
`SID`; others are meant to be secret and thus use the random `SID`.

For example, the hex numbers for the `xous-log` SID correspond to the ASCII string `b"xous-log-server "`, whereas
the `SID` for some of the `vault2` servers are just random numbers.

`xous-names` is a lookup service that can go from a descriptive, free-form `utf-8` string to a `SID`, should a server have
registered its `SID` with `xous-names` (not all `SID`s need to be registered with `xous-names`, and in fact servers
are free to exchange `SID`s directly for the highest level of undiscoverability). `xous-names` can implement a `TOFU`-style
access control, allowing a balance of discoverability and security.

## Interrupt Handlers

Interrupt handlers are dynamically registered by various services in Xous. Here's the list of handlers in this target:

```text
Interrupt handlers:
  IRQ | Process | Handler | Argument
    1:  usb-bao1x @ 2ff20 Some(20001980)
    2:  bao1x-hal-service @ 2a754 Some(20000758)
    5:  xous-log @ 19212 Some(20000008)
    8:  bao-video @ ad0dc Some(601f2d80)
    10:  bao1x-hal-service @ 23d4c Some(7ffffc2c)
    11:  keystore @ 25ace Some(20000008)
    20:  xous-ticktimer @ 1f7b4 Some(7ffffc30)
    30:  bao1x-hal-service @ 23cd0 Some(60009000)
```

The IRQ number corresponds to a hardware IRQ number (this is specific to the VexRiscv implementation). The `process` is the name of the process that has registered a handler for the IRQ; the hex number is the offset in process `.text` region where the handler code is located, and the `Argument` is the one 32-bit number that's passed to the interrupt handler that gives it some context to handle the interrupt.

## RAM Usage

This gives an idea of the RAM usage of various processes:

```text

RAM usage:
    PID   1:  160 k kernel
    PID   2:  364 k xous-swapper
    PID   3:   60 k keystore
    PID   4:   48 k xous-ticktimer
    PID   5:   64 k xous-log
    PID   6:   52 k xous-names
    PID   7:   68 k usb-bao1x
    PID   8:   84 k bao1x-hal-service
    PID   9:   48 k modals
    PID  10:  260 k pddb
    PID  11:   64 k bao-video
    PID  12:  244 k bao-console
    PID  13:  484 k vault2
2000 k total
```

This system has swap memory. Here you can see the swapper itself actually uses a good chunk of the available 2048kiB of memory in the system; all its pages have to stay resident so that it can manage memory faults without re-entrant swaps. PID's 1-11 in this system are XIP: they are written directly to RRAM, and thus the only RAM consumption is for the stack + heap. Also, processes that are not currently running have most of their data swapped out (such as `usb-bao1x` in this case).

`bao-console` and `vault2` are swap-located programs. These occupy a larger memory footprint in part because they are actively running and doing things, thus their working set is swapped in; but they also have a larger memory footprint in that any code that runs has to be RAM-resident because they are swap programs.
