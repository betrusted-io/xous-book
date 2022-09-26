# Xous Operating System Startup

The Xous operating system is set up by the loader, which is responsible for unpacking data into RAM and setting up processes. It is covered in the [Xous Loader](ch05-02-loader.md) section.

The loader reads a binary stream of data located in a tagged format that is discussed in the [Arguments Structure](ch05-01-arguments.md) section. This arguments structure defines features such as the memory layout, system configuration, and initial process data.

Programs are loaded in flattened formats called `MiniELF`, which is documented in the [MiniELF Format](ch05-03-minielf.md) section.

You may also find the following links of interest:
* [What happens before boot?](https://github.com/betrusted-io/betrusted-wiki/wiki/How-Does-Precursor-Get-to-the-Reset-Vector%3F) fills in the details of everything that happens before the first instruction gets run.
* "[Secure Boot](https://github.com/betrusted-io/betrusted-wiki/wiki/Secure-Boot-and-KEYROM-Layout)" and key ROM layout
