# MiniELF File Format

The loader uses a miniature version of the ELF file format.

ELF files support multiple sections. These sections have various flags, and may contain executable code, data, nothing, or debug information.

Traditional embedded programming relies on linker scripts to copy executable code into a format that can be programmed. Because Xous utilises an MMU, we can use ELF files natively.

A problem with the ELF format is that it contains a lot of overhead. The miniaturised version used here reduces the file size considerably while making it easier for the program to be loaded.

## Program Header

The program header contains just two pieces of information: The *load_offset*, and the *entrypoint*.

The *load_offset* is the offset, relative to the start of the Arguments structure, where various sections are stored. That is, if a section indicates that it is loading from address `0x100`, then the actual physical address can be calculated as:

* `0x100` * offset_of(arguments_list) + minielf.load_offset

The *entrypoint* is simply the value of the program counter when the program is first started.

## Section Headers

Following the Program Header is one or more Section Headers. The ELF format supports multiple section types, and does not have a fixed data/text/bss split, instead preferring a series of flags and values. The Xous image creation process opens the ELF file for each initial program and scans its section list. It skips any section that isn't required for running -- for example, symbol names, compile-time information, and debug information.

If a section is required for running and has no data -- for example if it's a `.bss` section -- then it sets the `NOCOPY` flag. Otherwise, data will get copied.

It then sets the `EXECUTE` and/or `WRITE` flags according to the ELF header.

Finally, it creates a new section entry in the Arguments structure with the specified flags, offset, and size. The offset used here is relative to the start of the output image on disk. Therefore, the very first section to be written will have an offset of `0`.

## ELF Flags

ELF supports multiple flags. For example, it is possible to mark a section as `Executable`, `Read-Only`, or `Read-Write`. Unfortunately these flags don't work well in practice, and issues can arise from various permissions problems.

Xous currently marks all pages `Read-Write-Execute`, however this may change in the future.

## Flattened MiniELF

The flattened MiniELF format is currently theoretical. This format would expand the on-disk representation of a process such that it was page-aligned. For example, in the storage format, offset `0x100` may be loaded to memory location `0x20000000`, while offset `0x110` may be loaded to offset `0x40000000`. The MMU is unable to create such fine-grained mappings, however a `Flattened MiniELF` file would reorder this such that the first memory location is stored at offset `0x1000` on the disk, allowing that entire page to be mapped to offset `0x20000000`. Padding will be added, and the subsequent data would be stored at offset `0x2000`. This allows the next page to be cleanly mapped to `0x40000000`.

This format will allow Execute-in-Place from SPI flash, which will free up additional memory.