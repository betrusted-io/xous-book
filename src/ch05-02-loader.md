# Xous Loader

The Xous loader is located in the [loader/](https://github.com/betrusted-io/xous/tree/main/loader) directory. This program runs in Machine mode, and makes the following assumptions:

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

## Reading Initial Configuration

The loader needs to know basic information about the Arguments structure before it can begin. This includes information about the memory layout, extra memory regions, kernel offset, and the number of initial programs.

The loader performs one pass through the Arguments structure to ensure that it contains the required fields before continuing.

## Loader Stage 1: Accounting

The first stage goes through the Arguments structure and does initial accounting. This involves multiple passes over the arguments structure.

### Runtime Page Tracker

The first pass sets up the *Runtime Page Tracker*. Each valid page in the system can be assigned to exactly one process. Memory that does not have an entry in the *Runtime Page Tracker* cannot be allocated, preventing us from allowing aliased memory.

Each page in main memory as well as each page in memory-mapped IO will get one byte of data in the *Runtime Page Tracker*. This byte indicates the process ID that the memory is assigned to. Process ID `0` is invalid, and indicates the page is free.

Whenever a page is allocated in the loader, it is marked in this region as belonging to the kernel -- i.e. PID 1. This region is passed to the kernel which will continue to use it to keep track of page allocations.

This memory is zeroed out and will be filled in later.

### Process Allocation

The loader allocates a set of initial processes, and it must pass this list of processes to the kernel. Fundamentally a process is just three things:

1. A memory space
2. An entrypoint
3. A stack

As such, the loader needs to allocate a table with these three pieces of information that is large enough to fit all of the initial processes. Therefore, it allocates a slice of memory that contains an `InitialProcess` struct that is big enough to cover all of the initial processes.

This structure is zeroed out, and will be filled in later.

### Argument Copying

The Arguments structure may be in RAM, but it may be located in some other area that will become inaccessible when the system is running. If configured, the Arguments structure is copied into RAM.

### Process Copying

Each process, plus the kernel, is then copied into RAM.

This is complex due to how memory data is laid out. For example, some sections are labeld NOCOPY, and indicate data such as `.bss` where there is no actual data to copy, it must simply be zeroed out.

### Setting page ownership

Mark all loader pages as being owned by `PID 1`. This ensures they cannot be reallocated later on.

## Loader Stage 2: Setting Page Tables

Now that memory has been copied, the second stage is responsible for parsing the loader file and setting up the system-specific page tables.

The loader walks the Arguments structure again and loops through each initial process as well as the kernel. For each process, it allocates the root page table, sets up the various memory sections with their requested permissions, allocates a stack, and marks all memory as loaded by the correct process.

After this is done, the loader maps all of the loader-specific sections into the kernel's memory space. In particular, the following are all mapped directly:

* Arguments structure
* Initial process list
* Runtime page tracker

## Jumping to the Kernel

The loader runs in Machine mode, which means the MMU is disabled. As soon as the loader jumps to the kernel, the CPU enters Supervisor mode with the MMU enabled and never again returns to Machine mode.

The loader stashes these settings in a structure called `backup_args`. This structure is currently placed at the end of loader stack, however in the future it may be allocated alongside structures such as the runtime page tracker.

Execution continues in `start_kernel`, which is located in `asm.S`.

In order to allow interrupts and exceptions to be handled by the kernel, the loader sets `mideleg` to `0xffffffff` in order to delegate all interrupts to Supervisor mode, and it sets `medeleg` to `0xffffffff` in order to delegate all CPU exceptions to the kernel.

The loader then does the handover by setting `mepc` and issuing a `reti` Return from Interrupt opcode.

## Resuming from Suspend

The operating system supports resuming from a cold poweroff. In order to get into this state, a program in the operating system wrote some values into RAM, then issued a command to power of the CPU in the middle of an interrupt handler.

A system is considered to be suspended when RAM contains a valid group of murmur3-signed hashes located at the 3rd page from the end of memory. If these hashes match, then the system is considered to be in suspend.

The loader then skips all remaining setup, because setup was previously performed and the system is in a live state. Indeed, if the loader tried to set up the data section of processes again, it would overwrite any volatile data in RAM.

In order to resume, the loader triggers a `STATE_RESUME` interrupt. This interrupt is not handled yet, since interrupts are not enabled. Instead, this interrupt will stay triggered until the kernel unmasks them, at which point the kernel will resume execution in the `susres` server and process the resume.

It then calls the kernel with arguments similar to a full boot. It reads values from the *backup_args* array located at the bottom of stack.

There is one change, however. Instead of beginning inside the kernel *main* function, the kernel begins executing immediately at the previous process. This causes the kernel to skip its initialization, and the kernel will resume where it left off once the preemption timer resumes.
