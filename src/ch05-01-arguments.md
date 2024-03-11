# System Arguments

The loader and kernel use a tagged format for defining system arguments. This tagged structure is designed to be small, and only describes data. The structure does not include any executable data. Instead, it contains references to this data that may be located immediately after the structure on a storage medium.

The tagged structure defines a prefix that is tagged by an 8-byte structure:

```rust
struct Tag {
    /// Ascii-printable name, not null-terminated, in little endian format.
    tag: u32,

    /// CRC16 of the data section, using CCITT polynomial.
    crc16: u16,

    /// Size of the data section, in 4-byte words.
    size: u16,
}
```

Tags are stored sequentially on disk, meaning a reader can skip over tags that it does not recognize. Furthermore, it can use a combination of `crc16` and `size` to determine that it has found a valid section.

The `size` field is in units of 4-bytes. Therefore, a `Tag` that contains only four bytes of data (for a total of 12-bytes on disk including the `Tag`) would have a `size` value of `1`.

## `XArg` tag -- Xous Arguments Meta-Tag

The only ordering requirement for tags is that the first tag should be an `XArg` tag. This tag indicates the size of the entire structure as well as critical information such as the size of RAM.

Future revisions may add to this tag, however the size will never shrink.


| Offset | Size | Name      | Description                                                                                                         |
| ------ | ---- | --------- | ------------------------------------------------------------------------------------------------------------------- |
| 0      | 4    | Arg Size  | The size of the entire args structure, including all headers, but excluding any trailing data (such as executables) |
| 4      | 4    | Version   | Version of the XArg structure.  Currently `1`.                                                                      |
| 8      | 4    | RAM Start | The origin of system RAM, in bytes                                                                                  |
| 12     | 4    | RAM Size  | The size of system RAM, in bytes                                                                                    |
| 16     | 4    | RAM Name  | A printable name for system RAM                                                                                     |

### `XKrn` tag -- Xous Kernel Description

This describes the kernel image.  There must be exactly one `XKrn` tag in an arguments structure.
This image will get mapped into every process within the final 4 megabytes, and therefore the text and data
offsets must be in the range `0xffc0_0000` - `0xfff0_0000`.

| Offset | Size | Name        | Description                                                                                                                              |
| ------ | ---- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 0      | 4    | LOAD_OFFSET | Physical address (or offset) where the kernel is stored                                                                                  |
| 4      | 4    | TEXT_OFFSET | Virtual memory address where the kernel expects the program image to live.  This should be `0xffd00000`.                                 |
| 8      | 4    | TEXT_SIZE   | Size of the text section.  This indicates how many bytes to copy from the boot image.                                                    |
| 12     | 4    | DATA_OFFSET | Virtual memory address where the kernel expects the .data/.bss section to be.  This should be above `0xffd00000` and below  `0xffe00000` |
| 16     | 4    | DATA_SIZE   | Size of the .data section                                                                                                                |
| 20     | 4    | BSS_SIZE    | The size of the .bss section, which immediately follows .data                                                                            |
| 24     | 4    | ENTRYPOINT  | Virtual address of the `_start()` function                                                                                               |

The kernel will run in Supervisor mode, and have its own private stack. The address of the stack will be generated by the loader.

### `IniE` and `IniF` tag -- Initial ELF Programs

The `IniE` and `IniF` tags describe how to load initial processes. There is one `IniE` or `IniF` for each initial program. There must be at least one `IniE` tag. `IniF` tagged processes are laid out in FLASH such that the sub-page offsets match 1:1 with their target virtual memory mapping. Thus, `IniF`-tagged processes are always memory-mapped from FLASH for XIP (execute in place) operation, where as `IniE` processes must be copied from disk into RAM and executed from the copy in RAM.

In other words, `IniE` processes consume more RAM because the static code base must be copied to RAM before execution. `IniF` processes consume less RAM, but not all processes can be `IniF`. In particular, the kernel and the FLASH management processes must be RAM based, because during FLASH write operations, the FLASH is unavailable for code execution reads.

This tag has the following values:

| Offset | Size | Name        | Description                                                                                                                             |
| ------ | ---- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| 0      | 4    | LOAD_OFFSET | Position in RAM relative to the start of the arguments  block where this program is stored, or an absolute value if `ABSOLUTE`  is `1`. |
| 4      | 4    | ENTRYPOINT  | Virtual memory address of the `_start()` function                                                                                       |

Following this is a list of *section definitions*. Section definitions must be sequential in RAM -- that is, it is not permitted for `SECTIONn_OFFSET` to decrease.

| Offset | Size | Name            | Description                                  |
| ------ | ---- | --------------- | -------------------------------------------- |
| n*3+8  | 8    | SECTIONn_OFFSET | Virtual memory address of memory section _n_ |
| n*3+12 | 3    | SECTIONn_SIZE   | Size of memory section _n_                   |
| n*3+15 | 1    | SECTIONn_FLAGS  | Flags describing memory section _n_          |

The fields `size`, `flags`, and `offset` together occupy 64 bits (8 bytes). The
`OFFSET` is a full 32-bit address.  The `SIZE` field is in units of
bytes, however as it is only 24 bits, meaning the largest section size
is `2^24` bytes. If these are printed as `u32` and read on the screen, the format
looks like this:

```text
0xJJJJ_JJJJ 0xKK_LLLLLL
```

Where:
  - `J` is the 32-bit offset
  - `K` is the 8-bit flag field
  - `L` is the 24-bit size field

#### `IniE` and `IniF` Flags

The `FLAGS` field contains the following four bits.  Any region may be
marked NOCOPY, however RISC-V does not allow regions to be marked
"Write-only":

| Bit | Binary   | Name       | Description                                   |
| --- | -------- | ---------- | --------------------------------------------- |
| 0   | 0b000001 | NOCOPY     | No data should be copied -- useful for `.bss` |
| 1   | 0b000010 | WRITABLE   | Region will be allocated with the "W" bit     |
| 2   | 0b000100 | READABLE   | Region will be allocated with the "R" bit     |
| 3   | 0b001000 | EXECUTABLE | Region will be allocated with the "X" bit     |
| 4   | 0b010000 | EH_FLAG    | Region is an EH_FLAG region                   |
| 5   | 0b100000 | EH_FLAG_HDR | Region is an EH_FLAG_HEADER region           |

These correspond 1:1 to the flag definitions used in the MiniELF format.

Programs **cannot** access the final four megabytes, as this memory
is reserved for the kernel. It is an error if any section enters this memory region.

### `PNam` Tag -- Program Names

`PNam` maps process IDs to process names. If multiple `PNam` tags exist
within a block, the first one that is encountered should take precedence.
This tag is a series of entries that take the following format:

| Size (bytes) | Name   | Description                                |
| ------------ | ------ | ------------------------------------------ |
| 4            | PID    | ID of the process that this name describes |
| 4            | Length | The length of the data that follows        |
| varies       | Data   | The UTF-8 name string                      |

### `Bflg` Tag -- Boot Flags

This configures various bootloader flags.  It consists of a single word
of data with various flags that have the following meaning:

* 0x00000001 `NO_COPY`  -- Skip copying data to RAM.
* 0x00000002 `ABSOLUTE` -- All program addresses are absolute.
  Otherwise, they're relative to the start of the config block.
* 0x00000004 `DEBUG`    -- Allow the kernel to access memory inside user
  programs, which allows a debugger to run in the kernel.

### `MREx` Tag -- Additional Memory Regions

This tag defines additional memory regions beyond main system memory. This region omits main system memory, which is defined in the `XArg` tag.
The format for this tag consists of a single word defining how many additional sections there are, followed by actual section entries:

| Offset | Size | Name  | Description                             |
| ------ | ---- | ----- | --------------------------------------- |
| 0      | 4    | Count | The number of additional memory entries |

Each additional memory entry is 3 words of 4-bytes each:

| Offset   | Size | Name   | Description                                                                        |
| -------- | ---- | ------ | ---------------------------------------------------------------------------------- |
| n*3 + 4  | 4    | Start  | The start offset of this additional region                                         |
| n*3 + 8  | 4    | Length | The length of this additional region                                               |
| n*3 + 12 | 4    | Name   | A 4-character name of this region that should be printable -- useful for debugging |

Additional memory regions should be non-overlapping. Creating overlapping memory regions will simply waste memory, as the loader will allocate multiple regions to track the memory yet will only allow it to be shared once.