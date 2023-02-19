# Xous Loader

The Xous loader is located in the [loader/](https://github.com/betrusted-io/xous-core/tree/main/loader) directory. This program runs in Machine mode, and makes the following assumptions:

1. There is an Argument structure located somewhere in memory and register `$a0` points to it
2. The system has 16 MB of RAM and it is located at address `0x40000000`

Point #2 is flexible, and the loader has the ability to read the memory configuration out of the Argument structure, if one can accept trusting these parameters before the Argument structure is checked. However, in the current implementation, these values are hard-coded into the loader binary so that they are derived from an already verified, trusted location (see Loader Signature Checking below for why this is the case).

After passing the signature check, the loader runs the main loader sequence. The loader runs in two stages. The first stage is responsible for determining how much memory is required for each initial process as well as the kernel, and loading them into memory. The second stage sets up the platform-specific page tables.

## Signature Checking the Kernel

We do not discuss precisely how we come to trust the loader itself: this responsibility falls onto a bootloader that is assumed to be burned into the ROM of the SoC running Xous. Please refer to [this page](https://github.com/betrusted-io/betrusted-wiki/wiki/How-Does-Precursor-Get-to-the-Reset-Vector%3F) for an example of one implementation for getting to the reset vector. It turns out in Precursor that the process to check the loader is identical to that of checking the kernel.

Loader conditions #1 and #2, as outlined above, are set up by the bootloader. The following context is helpful to appreciate why we hard-code the RAM address and offset instead of reading it out of the loader Arguments:

- The Arguments to the loader describe the location and size of Kernel objects, in addition to encoding the amount and location of RAM
- The loader and its Arguments are located in FLASH, so that it may be updated
- It is expensive and hard to update the loader's digital signature recorded in the SoC, as it is often burned to a bank of OTP fuses
- We assume that Kernel updates are routine, but loader updates are infrequent

Because the Arguments are tightly coupled to the Kernel image, we cannot check them at the same time that the loader binary. Therefore, we must treat the Arguments as untrusted at the entry point of the loader, and ask the loader to verify the Arguments. However, the loader needs to know its location and extent of RAM to run any Argument checking. Thus this presents a circular dependency: how are we to know where our memory is, when the structure that describes our memory is designed to be changed frequently? The method chosen to break this circular dependency is to hard-code the location and amount of RAM in the loader binary itself, thus allowing the Arguments that describe the kernel to be malleable with a signature check stored in FLASH.

Signatures for both the loader and the kernel share a common structure. They consist of two sections: the detached signature, and the signed data itself. The detached signature has the following format in memory:

| Offset | Size            | Name      | Description                                                                                                         |
| ------ | ----            | --------- | ------------------------------------------------------------------------------------------------------------------- |
| 0      | 4               | Version   | Version number of the signature record. Currently `1`                                                               |
| 4      | 4               | Length    | Length of the signed region (should be exactly +4 over the Length field in the signed region)                       |
| 8      | 64              | Signature | 64-byte Ed25519 signature of the signed region                                                                      |
| 12     | pad             | Padding   | 0-pad up to 4096 bytes                                                                                              |

The signed region has the following format:

| Offset            | Size              | Name      | Description                                                             |
| ------            | ----              | --------- | ----------------------------------------------------------------------- |
| 0                 | len(payload)      | Payload   | The signed payload (loader or kernel)                                   |
| len(payload)      | 4                 | Version   | A repeat of the version number of the signature record                  |
| len(payload)+4    | 4                 | Length    | len(payload) + 4 = length of all the data up to this point              |

Exactly every byte in the signed region, including the Version and Length, are signed. By including the Version and Length field in the signed region, we can mitigate downgrade and length extension attacks.

Signatures are computed using the [Dalek Cryptography Ed25519](https://github.com/dalek-cryptography/ed25519-dalek) crate.

The public key used to check the signature can come from one of three sources:

1. A self-generated key. This is the "most trusted" source. Ultimately, every device should self-sign its code.
2. A third-party key. We do not handle the thorny issue of who provides the third party key, or how we come about to trust it.
3. A developer key. This is a "well known" key which anyone can use to sign an image.

The loader will attempt to verify the kernel image, in sequence, with each of the three keys. If it fails to find any image that matches, it prints an error message to the display and powers the system down after a short delay.

If the image is signed with anything but the self-generated key, a visible marker (a set of fine dashed lines over the status bar) is turned on, so that users are aware that there could be a potential trust issue with the boot images. This can be rectified by re-computing a self-signature on the images, and rebooting.

Upon the conclusion of the signature check, the loader also does a quick check of the stack usage, to ensure that nothing ran out of bounds. This is important because the Kernel assumes that no memory pages are modified across a suspend/resume, except for the (currently) two pages of RAM allocated to the loader's stack.

## Loading the OS

Once the image has been signature checked, the loader must set up the Xous kernel. Xous has a very regular structure, where everything is a process, including the kernel. What makes the kernel special is that it is process ID `1`, and its code is also mapped into the high 4 MiB of every other processes' memory space, allowing processes to run kernel code without having to swap out the `satp` (that is, the page table base).

The loader's responsibility is to go from a machine that has essentially a zero-ized RAM space and a bunch of archives in FLASH, to one where physical pages of memory are mapped into the correct virtual address spaces for every process.

This is done in several stages:

1. Reading the configuration
2. `Stage 1` - copying processes to their final runtime locations, and keeping track of all the copies
3. `Stage 2` - creating a page table that reflects the copies done in `Stage 1`
4. Jumping to the kernel

### Reading Initial Configuration

The loader needs to know basic information about the Arguments structure before it can begin. This includes information about the memory layout, extra memory regions, kernel offset, and the number of initial programs.

The loader performs one pass through the Arguments structure to ensure that it contains the required fields before continuing.

### Loader Stage 1: Copying and Aligning Data

Stage 1 copies and aligns all of the processes, such that the sub-page offsets for the code matches the expectations that the linker set up. It also copies any data requires write access, even if is already correctly aligned. The core routine is `copy_processes()`.

In the case that the offsets for the memory image on FLASH line up with the virtual memory offsets, nothing needs to be done for the text, read-only data, and exception handler sections. In the case that they do not line up, a copy must be made in RAM of these sections to ensure correct alignment.

Virtual sections marked as `NOCOPY` must be allocated and zero-ized, and sections marked with write access must always the copied.

The loader reserves the top two pages for its own working stack space, and adds a configurable `GUARD_MEMORY_BYTES` buffer to determine the beginning of process space. Note that one page of the "guard" area is used to store a "clean supend marker", which is used by the loader to check if the current power-on cycle is due to a resume, or a cold boot.

Currently, the total works out to an offset of 16kiB reserved from top of RAM before processes are copied. These RAM pages are "lost forever" and not available to the Xous kernel for any purpose. This physical offset represents the start of the loader's workspace for setting up Xous.

#### Process Copying

The loader consumes RAM, page-by-page, starting from the highest available physical offset, working its way down.

The loader iterates through the list of user process images (`IniE`/`IniF`), and copies data into physical pages of RAM based on the following set of rules:

1. If a new page is required, decrement the "top of RAM" pointer by a page and zeroize the page
2. "If necessary", copy (or allocate) the section from FLASH to RAM, taking care to align the target data so that it matches the expected virtual address offset
3. Zeroize any unallocated data in the page

The "if necessary" rules are as follows:

1. If it is an `IniE`, always copy or allocate the sections. Sections marked as `NOCOPY` are simply allocated in virtual memory and zeroized.
2. If it is an `IniF`, only copy sections marked as writeable; allocate any `NOCOPY` areas.

At the conclusion of process copying, the "top of RAM" pointer is now at the bottom of all the physical pages allocated for the user processes.

At this point, the kernel is copied into RAM, and the top of RAM pointer is decremented accordingly. The algorithm is very similar to the above except there are fewer sections to deal with in the kernel.

Finally, the "top of RAM" is located at the lowest address in RAM that has been allocated for the initial process state of Xous.

### Loader Stage 2: Setting Page Tables

Now that memory has been copied, the second stage is responsible for re-parsing the loader file and setting up the system-specific page tables. To recap, memory is laid out as follows:

- Top of RAM
- Loader stack (2 pages currently)
- Guard area (2 pages; one page used for "clean suspend marker")
- PID2
- PID3
- ...
- Kernel
- Pointer to next free memory

However, the page tables have not been set up.

Stage 2 iterates through the arguments in the same order as "Stage 1", except this time we know where the "bottom" of total physical memory is, so we know where to start allocating pages of the page table.

Thus, Stage 2 walks the Arguments structure again, in the exact same order as Stage 1. For each process, it allocates the root page table (noting the SATP location), sets up the various memory sections with their requested permissions, allocates a default stack, and marks all memory as loaded by the correct process in the `runtime page tracker`, discussed in the next subsection. This is done using a series of `alloc` routines that simply decrement the "top of RAM" pointer and hand back pages to be stuck into page table entries.

Note that "allocating stack" simply refers to the process of reserving some page table entries for would-be stack; the actual physical memory for stack isn't allocated until runtime, when a page fault happens in stack and the kernel grabs a physical page of RAM and slots it into the stack area on demand.

After this is done, the loader maps all of the loader-specific sections into the kernel's memory space. In particular, the following are all mapped directly into the kernel's memory space:

* Arguments structure
* Initial process list
* Runtime page tracker

#### Runtime Page Tracker

The *Runtime Page Tracker* is a slice of `PID`, where each entry corresponds to a page of physical RAM or I/O, starting from the lowest RAM available to the highest RAM, and then any memory-mapped I/O space. Because the loader doesn't have a "heap" per se to allocate anything, the runtime page tracker is conjured from a chunk of RAM subtracted from the "top of free RAM" pointer, and then constructed using `slice::from_raw_parts_mut()`. This slice is then passed directly onto the kernel so it has a zero-copy method for tracking RAM allocations.

Thus, the *Runtime Page Tracker* is a whitelist where each valid page in the system can be assigned to exactly one process. Memory that does not have an entry in the *Runtime Page Tracker* cannot be allocated. This helps prevent memory aliasing attacks in the case that a hardware module does not fully decode all the address bits, because RAM that isn't explicitly described in the SVD description of the SoC can't be allocated.

Thus, the image creation program must be passed a full SVD description of the SoC register model to create this whitelist. It shows up as the "Additional Regions" item in the Xous Arguments output as displayed by the image creation program.

Each page in main memory as well as each page in memory-mapped IO will get one byte of data in the *Runtime Page Tracker*. This byte indicates the process ID that the memory is assigned to. Process ID `0` is invalid, and indicates the page is free.

Whenever a page is allocated in the loader, it is marked in this region as belonging to the kernel -- i.e. PID 1. This region is passed to the kernel which will continue to use it to keep track of page allocations.

#### Process Allocation

The loader allocates a set of initial processes, and it must pass this list of processes to the kernel. Fundamentally a process is just three things:

1. A memory space
2. An entrypoint
3. A stack

As such, the loader needs to allocate a table with these three pieces of information that is large enough to fit all of the initial processes. Therefore, in a manner similar to the *Runtime Page Tracker*, it allocates a slice of memory that contains an `InitialProcess` `struct` that is big enough to cover all of the initial processes.

This structure is zeroed out, and is filled in by the Stage 2 loader.

### Preparing to Boot

At this point, RAM looks something like this:

- Top of RAM
- Loader stack (2 pages currently)
- Guard area (2 pages; one page used for "clean suspend marker")
- PID2
- PID3
- ...
- Kernel
- Runtime page tracker
- Initial process table, holding pointers to root page table locations
- Page table entries for PID2
- Page table entries for PID3
- ...
- Page table entries for the kernel

The kernel is now ready for the pivot into virtual memory, so we perform the following final steps.

#### Argument Copying

The Arguments structure may be in RAM, but it may be located in some other area that will become inaccessible when the system is running. If configured, the Arguments structure is copied into RAM.

#### Setting page ownership

Mark all loader pages as being owned by `PID 1`. This ensures they cannot be reallocated later on and overwritten by a rogue process.

## Jumping to the Kernel

The loader runs in Machine mode, which means the MMU is disabled. As soon as the loader jumps to the kernel, the CPU enters Supervisor mode with the MMU enabled and never again returns to Machine mode.

The loader stashes these settings in a structure called `backup_args`. This structure is currently placed at the end of loader stack, however in the future it may be allocated alongside structures such as the runtime page tracker.

Execution continues in `start_kernel`, which is currently located in `asm.S` (but is slated to migrate into a `.rs` file now that assembly is part of stable Rust).

In order to allow interrupts and exceptions to be handled by the kernel, which runs in Supervisor mode, the loader sets `mideleg` to `0xffffffff` in order to delegate all interrupts to Supervisor mode, and it sets `medeleg` to `0xffffffff` in order to delegate all CPU exceptions to the kernel.

The loader then does the handover by setting `mepc` (the exception return program counter - contains the virtual address of the instruction that
nominally triggered the exception) to the `entrypoint` of the kernel, and issuing a `reti` (Return from Interrupt) opcode.

Thus one can effectively think of this entire "boot process" as just one big machine mode exception that started at the reset vector, and now, we can return from this exception and resume in Supervisor (kernel) code.

## Resuming from Suspend

The operating system supports resuming from a cold poweroff. In order to get into this state, a program in the operating system wrote some values into RAM, then issued a command to power of the CPU in the middle of an interrupt handler.

A system is considered to be suspended when RAM contains a valid group of murmur3-signed hashes located at the 3rd page from the end of memory. If these hashes match, then the system is considered to be in suspend.

The loader then skips all remaining setup, because setup was previously performed and the system is in a live state. Indeed, if the loader tried to set up the data section of processes again, it would overwrite any volatile data in RAM.

In order to resume, the loader triggers a `STATE_RESUME` interrupt. This interrupt is not handled yet, since interrupts are not enabled. Instead, this interrupt will stay triggered until the kernel unmasks them, at which point the kernel will resume execution in the `susres` server and process the resume.

It then calls the kernel with arguments similar to a full boot. It reads values from the *backup_args* array located at the bottom of stack.

There is one change, however. Instead of beginning inside the kernel *main* function, the kernel begins executing immediately at the previous process. This causes the kernel to skip its initialization, and the kernel will resume where it left off once the preemption timer resumes.
