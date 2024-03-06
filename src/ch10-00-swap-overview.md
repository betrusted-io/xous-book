# Encrypted Swap

Encrypted swap is a solution for small-footprint secure microcontrollers that must rely upon external RAM chips. The idea is to have the fast, on-chip RAM within the secure perimeter serve as the working set of data, but have this backed up with a "swapfile" consisting of slower, off-chip RAM that is also encrypted.

The swap implementation is modelled after the kind of swap space found in other operating systems like Linux. The kernel can over-commit pages of virtual memory, and data is only allocated when a program actually attempts to access the over-committed pages. When the on-chip memory gets full, the swapper will guess as to what pages are not being used and copy them to encrypted swap. The kernel is then free to mark those virtual memory pages as invalid, and re-allocate them to actively used data.

Terminology:

- `wired` refers to pages that are resident in RAM and cannot be swapped out
- `resident` refers to a page in RAM
- `free RAM` refers unallocated internal RAM
- `swapped` refers to a page in swap
- `free swap` refers to unallocated pages in external swap memory
- `reserved` refers to memory that has been over-committed, i.e., exists in virtual memory space but has no physical allocation
- `allocated` refers to memory that has been allocated, but could be in either `resident` or `swapped` states

## Review of Virtual Memory Implementation Without Swap

In Xous, every process has a page table, including the kernel. The `satp` field of a process record stores the page address of the root of a process page table in `satp.PPN`. `satp.ASID` contains the PID.

The kernel is mapped into every process virtual memory space, in the top 16MiB of memory. Thus, the kernel is unique in that it is the only process that can access its physical pages alongside another process.

When each process is created, their page tables are populated with entries that hard-wire the program's code. The stack and heap are also fully allocated, but over-provisioned. Only the first page of each space is backed with physical memory; the rest is demand-paged. Thus, when a program starts, its maximum stack and heap extents are defined by the loader. These extents can be modified at runtime with a kernel syscall, but a program will OOM-fail even if there is physical memory available if its virtual stack and heap allocations are exhausted.

To review, each PTE has the following flags:

- V `valid`: page contents are valid
- R `read`: page can be read
- W `write`: page can be written
- X `execute`: page is executable
- U `user`: page is accessible in user mode
- G `global`: page is in all address spaces
- A `accessed`: page has been accessed (read, execute, or write) (Vex does not implement)
- D `dirty`: page has been written since last time it was cleared (Vex does not implement)
- S `shared`: (part of RWS set) shared page - used by Xous to track if a page has been lent to another process
- P `previously writable`: (part of RWS set) previously writable (not used)

From the standpoint of memory management, a page can only have the following states:

- [Allocated](https://github.com/betrusted-io/xous-core/blob/f389b41ccf3f31d4565b6840403af522ffc16890/kernel/src/arch/riscv/mem.rs#L894-L896): `V` set
- [Fault](https://github.com/betrusted-io/xous-core/blob/f389b41ccf3f31d4565b6840403af522ffc16890/kernel/src/arch/riscv/mem.rs#L901-L903): No flags are set, or `S` is set and `V` is not set
- Reserved: `V` is *not* set, and at least one other flag is set except for `S`

## Encryption Method

Swap is encrypted using AES-GCM-SIV. The swap encryption key is generated from a hardware TRNG on boot. It is critical that this TRNG function correctly and be truly random early at boot.

The 16-byte AES-GCM-SIV MAC codes for every page are stored in a global appendix in untrusted RAM; this is fine, as the MAC is considered to be ciphertext, as all security derives from the key, but the swap space is reduced by this overhead.

The nonce for AES-GCM-SIV is derived as follows:

`nonce[96] = {swap_count[32]|pid[8]|p_page[24]|v_page[24]}`

This gives the following security properties:
- In all cases, pages cannot be replayed between reboots, as the key is generated on each boot
- Tampered pages in external memory are cryptographically likely to be detected (due to the 128-bit MAC code)
- Identical plaintext between two processes will not map to the same ciphertext
- Identical plaintext within a process will not map to the same ciphertext
- Ciphertext between two processes cannot be swapped to do code injection with ciphertext
- Ciphertext of one page within a process cannot be copied to a new page location in the same process and decrypt correctly
- Identical plaintext located in the same physical location and virtual location for the same process with the same swap count will create interchangeable ciphertext

The swap count of a page is a cryptographically small number (nominally 32 bits) that is used to track which page has been least-recently used (to manage evictions). The implementation will be coded to allow a larger number if necessary, but there is a trade-off between the size of this number and the amount of heap space needed to track every virtual page and its swap space; as the swap grows large, the overhead can start to overwhelm the amount of memory available in a small footprint microcontroller.

The last property means that, for example, it is possible to infer something about the heap or stack of a running process that has been swapped out. In particular, we can detect if a region of stack or heap memory has been modified & swapped, and then restored to the original data & swapped, once the swap count is saturated.

This can be used for the following malicious activities:
  - Build a side channel to exfiltrate data via swap
  - Force a running process into a previous stack or heap configuration by recording and restoring pages after forcing the swap count to roll over

Mitigation of this vulnerability relies upon these factors:
  - It takes a long time to push the swap count to 32 bits. This is the equivalent of encrypting and decrypting about 17 terabytes of data using the embedded controller. This would take over 22 years if the microcontroller can run AES-GCM-SIV bidirectionally at a rate of 100MiB/s. An Apple M1 Pro [can achieve 590MiB/s](https://engineering.linecorp.com/en/blog/AES-GCM-SIV-optimization) unidirectionally, so this is an optimistic estimate for an embedded controller running at a few hundred MHz.
  - Instead of saturating, the swap count must roll-over. This means that the LRU algorithm will suffer a performance degradation after a "long time" (see previous bullet for bounds on that), but by avoiding saturation it means an attacker has a single window of opportunity to re-use a page before having to roll over again (or they must keep a log of all the pages). This isn't a cryptographically strong defense, but it practically complicates any attack with little cost in implementation.
  - The swap count can be increased to up to 40 bits in size (any larger would overflow the nonce size considering the other data concatenated into the nonce), if further strength is desired, at a considerable price on 32-bit microcontrollers.

## Swap Implementation

Swap is a kernel feature that is enabled with the flag `swap`.

Swap is intended to be implemented using an off-chip SPI RAM device. While it is most performant if it is a memory mapped RAM, it does not need to be; the `swapper` shall be coded with a HAL that can also handle SPI devices that are accessible only through a register interface.

### Boot Setup: Loader

The loader gets new responsibilities when `swap` is enabled:
- The loader needs to be aware of both the location and size of the trusted internal unencrypted RAM (resident memory), and the external encrypted RAM (swap memory).
- The resident memory is tracked using the existing "Runtime Page Tracker" (RPT) mechanism.
- Two additional structures are created:
   - The "Swap Page Tables" (SPT), which is a slice of pointers to swap page table structures, one for each PID that is using swap.
   - The "Swap MAC Table" (SMT), which tracks the 16-byte MAC codes for every page in swap. It does not degrade security to locate the SMT in swap.
- The "Swap Count Tracker" is not allocated by the loader. However, the swap count of pages in swap is guaranteed to be set to 0 by the loader.
- The loader is responsible for querying the TRNG on every boot to generate the session key for encrypting off-chip RAM.
- A new image tag type is created `inis`, to indicate data that should start in encrypted swap.

The SPT has the same structure as system page tables. However, SPT entries are only allocated on-demand for processes that have swap; it is not a fully copy of every page in the system page table.

#### INIF handling

Regions marked as `xip` (i.e, marked as `inif` type) are assumed to FLASH that is contained within the secure perimeter of the microcontroller. These regions are mapped directly and unencrypted into the kernel memory space.

These boot-critical processes must be `xip`, and are never swapped out:

- kernel
- swapper
- ticktimer

More regions can be marked as `xip`; this is preferable, because if the code is resident in on-chip FLASH, they aren't taking up working-set RAM and things will run faster.

The initial data set of `inif` processes are considered to be `wired`; however, their reserved memory regions can be swapped after allocation.

#### INIE handling

Regions marked as `inie` are copied to the working set RAM and executed out of RAM. It is assumed this tag is mostly unused in microcontrollers with a small internal working set, but the behavior is not modified because it is a valid and useful tag for devices with large, trusted RAMs.

#### INIS handling

Regions marked as `inis` are copied into encrypted swap on boot. The kernel page table state start out with the correct `R`/`W`/`X`/`U` values, but `V` is not set, and `P` is set. Entries are created in the SPT to inform the tracker where to find the pages.

An kernel argument of type `swap` is provided, which is a base and bounds to the SPT/SMT region. This is meant to passed to the `swapper` process when it registers to the kernel.

The image creation routine and kernel arguments need to be extended to support `inis` regions located in off-chip SPI FLASH. The off-chip data is not encrypted, but it is signature checked with a dedicated signature block. Note that the off-chip SPI FLASH does not need to be memory mapped: the loader may read the memory through a register interface.

### Kernel Runtime

Systems with `swap` enabled must have a process located at `PID` 2 that is the `swapper`. The kernel will only recognize `swap` extension syscalls originating from `PID` 2.

The following kernel syscall extensions are recognized when the `swap` feature is activated:

- `RegisterSwapper`
- `EvictPage`

The `swapper` must also implement the following opcodes:

- `WriteToSwap`: a memory message that copies & encrypts the mapped page to swap. The `offset` & `valid` fields encode the original PID and virtual address.
- `ReadFromSwap`: a memory message that retrieves & decrypts a page from swap, and copies it to the lent page. The `offset` & `valid` fields encode the original PID and virtual address.
- `AllocateAdvisory`: a scalar message that informs the swapper that a page in free RAM was allocated to a given PID and virtual address
- `Trim`: a request from the kernel to free up N pages. Normally the kernel would not call this, as the swapper should be pre-emptively clearing space, but it is provided as a last-ditch method in case of an OOM.
- `Free`: a scalar message that informs the swapper that a page was de-allocated by a process.

#### Flags and States

When `swap` is enabled, the flags have the following meaning:

- V `valid`: page contents are valid
- R `read`: page can be read
- W `write`: page can be written
- X `execute`: page is executable
- U `user`: page is accessible in user mode
- G `global`: page is in all address spaces
- A `accessed`: page has been accessed (read, execute, or write) (Vex does not implement)
- D `dirty`: page has been written since last time it was cleared (Vex does not implement)
- S `shared`: (part of RWS set) shared page - used by Xous to track if a page has been lent to another process
- P `swapped`: (part of RWS set) page is in swap

From the standpoint of memory management, a page can only have the following states:

- `Allocated`: `V` set, `P` may not be set
- `Fault`: No flags are set, or `S` is set and `V` is not set
- `Swapped`: `P` is set. `V` is *not* set. `S` may also not be set. Upon access to this page, the kernel allocates a resident page and calls `ReadFromSwap` to fill it. The page will move to the `Allocated` state on conclusion.
- `Reserved`: `V` is *not* set, `P` is *not* set, and at least one other flag is set except for `S`. A kernel allocates a resident page and zeros it. The page will move to the `Allocated` state on conclusion.

Pages go from `Allocated` to `Swapped` based on the `swapper` observing that the kernel is low on memory, and calling a series of `EvictPage` calls to free up memory. It is always assumed that the kernel can allocate memory when necessary; as a last ditch the kernel can attempt to call `Trim` on the swapper, but this should only happen in extreme cases of memory pressure.

Pages go from `Allocated` to `Reserved` when a process unmaps memory.

When the `swapper` runs out of space, `WriteToSwap` panics with an OOM.

#### RegisterSwapper Syscall

The `swapper` registers with the kernel on a TOFU basis. The kernel reserves a single 128-bit `sid` with the target of the `swapper`, and it will trust the first process to use the `RegisterSwapper` syscall with its 128-bit random ID.

After registration, the kernel sends a message to the `swapper` with the location of the SPT/SMT regions as created by the bootloader, as well as the base and bounds of the free memory pool. The free memory pool is the region remaining after boot, after the loader has marked all the necessary RAM pages as `wired`.

#### EvictPage Syscall

`EvictPage` is a syscall that only the `swapper` is allowed to call. It is a scalar `send` message, which contains the PID and address of the page to evict. Upon receipt, the kernel will:

- Change into the requested PID's address space
- Lookup the physical address of the evicted page
- Clear the `V` bit and set the `P` bit of the evicted page's PTE
- Change into the swapper's address space
- Mutably lend the evicted physical page to the swapper with a `WriteToSwap` message
- Schedule the swapper to run

### Swapper Responsibilities

It is the swapper's responsibility to maintain a structure that keeps track of every page in the `free RAM` pool. It builds this using the `AllocateAdvisory` messages.

When the amount of free RAM falls below a certain threshold, the swapper will initiate a `Trim` operation. A kernel can also initiate a `Trim` as a last-ditch in case the `swapper` was starved of time and was unable to initiate a `Trim`, but this meant to be avoided.

In a `Trim` operation, the swapper picks the pages it thinks will have the least performance impact and calls `EvictPage` to remove them. Initially, the swapper will have no way to know what pages are most important; it must use a heuristic to guess the first pages to evict. However, since it must track how frequently a page has been swapped with the `swap_count` field (necessary for encryption), it can rely upon this to build an eventual map of which pages to avoid.

The swapper is also responsible for responding to `WriteToSwap` and `ReadFromSwap` calls.

