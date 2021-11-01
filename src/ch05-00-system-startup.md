# Xous Operating System Startup

The Xous operating system is set up by the loader, which is responsible for unpacking data into RAM and setting up processes. It is covered in the [Xous Loader](ch05-02-loader.md) section.

The loader reads a binary stream of data located in a tagged format that is discussed in the [Arguments Structure](ch05-01-arguments.md) section. This arguments structure defines features such as the memory layout, system configuration, and initial process data.

Programs are loaded in flattened foramts called `MiniELF`, which is documented in the [MiniELF Format](ch05-03-minielf.md) section.