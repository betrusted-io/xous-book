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

Xous supports a subset of these flags, as documented in the [flags section](ch05-01-arguments.md#inie-and-inif-flags) of the `IniE` and `IniF` tags.

## Page-Aligned MiniELF

Programs that are meant to be run out of FLASH directly and not copied to RAM are laid out in the MiniELF archive format such that the sub-page address offsets (that is, the lower 12 bits) correspond 1:1 with the virtual memory mappings. This allows the FLASH copy of the MiniELF to be simply mapped into the correct location in virtual memory for the target process, instead of being copied into RAM. `IniF`-tagged sections comply to this discipline.

The penalty for page-aligned MiniELF is minor; primarily, a few bytes of padding have to be inserted here and there as the files are generated to ensure that the alignment requirements are met. The main overhead is the cognitive load of peeking into the next iteration of a Rust iterator to determine what the alignment of the *next* section should be so that you can finalize the padding of the *current* section.

However, the `IniE` format is retained as-is for historical reasons.
