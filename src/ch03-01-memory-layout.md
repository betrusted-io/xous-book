# Memory Management in Xous

Memory is allocated with the `MapMemory` syscall. This call accepts four arguments:

* physical: `Option<NonZeroUsize>`: The physical address you would like to allocate. Specify `None` if you don't need a particular address.
* virtual: `Option<NonZeroUsize>`: The virtual address you would like to allocate. Specify `None` if you don't need a particular virtual address.
* size: `NonZeroUsize`: The size of the region to allocate. This must be page-aligned.
* flags: `MemoryFlags`: A list of platform-specific flags to apply to this region.

The memory will return a `MemoryRange` that encompasses the given region.

You can free memory with `UnmapMemory`, though be very careful not to free memory that is currently in use. `UnmapMemory` simply takes the `MemoryRange` returned by `MapMemory`.

## Physical Addresses

A program rarely needs to access physical addresses, and in most operating systems it's not the kind of thing you can actually *do*. However, Xous is designed to be embedded, so it's entirely legal to request a physical address.

The trick is that you can only request physical addresses that actually exist. For example, you cannot request a physical address for a mirrored region of a peripheral because that is not a valid address.

**If you request a physical address from main RAM, the memory will be zeroed when you receive it**. Peripherals and ares that are not in main RAM will not be zeroed. It is for this reason that system services are recommended to claim all peripherals before running user programs.

## Virtual Addresses

All Xous programs run with virtual memory enabled. Attempting to perform an illegal operation will result in an exception. If you have an exception handler installed, illegal memory accesses will run this exception handler which may fix up the exception.

### Demand Paging

When you allocate memory using `MapMemory(None, None, ..., ...)`, you will be handed memory from the `DEFAULT_BASE`. This memory will not be backed by a real page, and will only be allocated by the kernel once you access the page. This allows threads to allocate large stacks without running out of memory immediately.

Pages that are mapped-but-unallocated are visible in a process' page table view. As an example, consider the following excerpt from a page table view:

```text
    38 60026000 -> 400a3000 (flags: VALID | R | W | USER | A | D)
    41 60029000 -> 40108000 (flags: VALID | R | W | A | D)
    42 6002a000 -> 40109000 (flags: VALID | R | W | A | D)
    43 6002b000 -> 00000000 (flags: R | W)
    44 6002c000 -> 00000000 (flags: R | W)
```

Addresses 0x60026000 (#38), 0x60029000 (#41), and 0x6002a000 (#42) are all allocated. The rest of the pages are valid-but-unallocated.

Address 0x60026000 (#38) is mapped to the process and has a valid physical address. Reads and writes to this page are backed by physical address 0x400a3000.

Addresses 0x60029000 (#41) and 0x6002a000 (#42) are still owned by the kernel, likely because they were being cleared.

Addresses 0x6002b000 (#43) and 0x6003c000 (#44) are on-demand allocated. They have no physical backing, and attempting to access them will result in a kernel fault where they will be allocated. When the page is allocated, it will be given the flags `R | W` in addition to default kernel flags.

### The Heap

When we talk about "The Heap" we mean data that is managed by functions such as `malloc`. Xous has a pair of syscalls that behave vaguely like the Unix `brk` command.

`IncreaseHeap(usize, MemoryFlags)` will increase a program's heap by the given amount. This returns the new heap as a `MemoryRange`.

To decrease the heap by a given amount, call `DecreaseHeap(usize)`.

Note that you must adjust the heap in units of `PAGE_SIZE`.

You can avoid using these syscalls by manually allocating regions using `MapMemory`, however they are a convenient abstraction with their own memory range.

`liballoc` as bundled by Xous uses these syscalls as a backing for memory.

### Virtual Memory Regions

There are different memory regions in virtual address space:

| Address    | Name | Variable | Description
| ---------- | ---- | -------- | -----------
| 0x0001_0000 | text | - | Start of `.text` with the default riscv linker script (`riscv64-unknown-elf-ld -verbose`)
| 0x2000_0000 | heap | DEFAULT_HEAP_BASE | Start of the heap section returned by `IncreaseHeap`
| 0x4000_0000 | message | DEFAULT_MESSAGE_BASE | Base address where `MemoryMessage` messages are mapped inside of a server
| 0x6000_0000 | default | DEFAULT_BASE | Default region when calling `MapMemory(..., None, ..., ...) -- most threads have their stack here
| 0x7fff_ffff | stack   | - | The default stack for the first thread - grows downwards from 0x8000_0000 not inclusive
| 0xa000_0000 | swhal   | SWAP_HAL_VADDR | Hardware-specific pages for the swapper. For configs that use memory-mapped swap, contains the memory mapping (and thus constrains total swap size). For configs that use register-mapped swap, contains the HAL structures for the register driver. These configurations could potentially have effectively unlimited swap.
| 0xe000_0000 | swpt    | SWAP_PT_VADDR | Swap page table roots. One page per process, contains virtual addresses (meant to be walked with code)
| 0xe100_0000 | swcfg   | SWAP_CFG_VADDR | Swap configuration page. Contains all the arguments necessary to set up the swapper.
| 0xe100_1000 | swrpt   | SWAP_RPT_VADDR | Location where the memory allocation tracker (runtime page tracker) is mapped when it is shared into userspace.
| 0xe110_0000 | swcount | SWAP_COUNT_VADDR | Location of the block swap count table. This is statically allocated by the loader before the kernel starts.
| 0xff00_0000 | kernel  | USER_AREA_END | The end of user area and the start of kernel area
| 0xff40_0000 | pgtable | PAGE_TABLE_OFFSET | A process' page table is located at this offset, accessible only to the kernel
| 0xff80_0000 | pgroot  | PAGE_TABLE_ROOT_OFFSET | The root page table is located at this offset, accessible only to the kernel
| 0xff80_1000 | process | PROCESS | The process context descriptor page
| 0xffc0_0000 | kargs   | KERNEL_ARGUMENT_OFFSET | Location of kernel arguments
| 0xffd0_0000 | ktext   | - | Kernel `.text` area. Mapped into all processes.
| 0xfff7_ffff | kstack  | - | Kernel stack top, grows down from 0xFFF8_0000 not inclusive
| 0xfffe_ffff | exstack | - | Stack area for exception handlers, grows down from 0xFFFF_0000 not inclusive


In addition, there are special addresses that indicate the end of a function. The kernel will set these as the return address for various situations, and they are documented here for completeness:

| Address    | Name | Variable | Description
| ---------- | ---- | ------ | -----------
| 0xff80_2000 | retisr  | RETURN_FROM_ISR | Indicates the return from an interrupt service routine
| 0xff80_3000 | exitthr | EXIT_THREAD | Indicates a thread should exit
| 0xff80_4000 | retex   | RETURN_FROM_EXCEPTION_HANDLER | Indicates the return from an exception handler
| 0xff80_8000 | retswap | RETURN_FROM_SWAPPER | Indicates the return from the userspace swapper code. Only available when `swap` feature is selected.
