# Xous Image Creation

Xous image creation is primarily performed by the `create-image` program. This program bundles memory definitions, the kernel, and initial programs together and generates an image on-disk suitable for passing to the loader.

You can run this program manually to see how it works:

```sh
$ cargo run -p tools --bin create-image -- --help
    Finished dev [unoptimized + debuginfo] target(s) in 0.19s
     Running `target/debug/create-image --help`
Xous Image Creator 0.1.0
Sean Cross <sean@xobs.io>
Create a boot image for Xous

USAGE:
    create-image [FLAGS] [OPTIONS] <OUTPUT> --csv <CSR_CSV> --kernel <KERNEL_ELF> --ram <OFFSET:SIZE> --svd <SOC_SVD>

FLAGS:
    -d, --debug      Reduce kernel-userspace security and enable debugging programs
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -c, --csv <CSR_CSV>          csr.csv file from litex
    -i, --init <init>...         Initial program to load
    -k, --kernel <KERNEL_ELF>    Kernel ELF image to bundle into the image
    -r, --ram <OFFSET:SIZE>      RAM offset and size, in the form of [offset]:[size]
    -s, --svd <SOC_SVD>          soc.csv file from litex

ARGS:
    <OUTPUT>    Output file to store tag and init information
$
```

This program generates an [Arguments structure](ch05-01-arguments.md) based on the specified commands, and copies data from the given ELF files into an area immediately following this structure. In this manner, a complete, position-independent loadable system is generated in a single binary image.

This program also does rudimentary sanity checking. For example, it will ensure the kernel is loaded at a sane offset -- namely above address `0xff000000`. It will also ensure the memory regions don't overlap.

As a special case, it will trim the CSR section down from the reported size. In essence, while the configuration region is defined as 256 MB wide, this large region is never used in practice. In order to reduce the amount of memory required to store this data, as well as in order to remove memory aliasing attacks, the CSR region is trimmed down from the reported value to only encompass ranges that are valid.

## The Kernel and Initial Programs

The kernel, as well as each initial program, are all packaged together in the same `args.bin` file. In order to do this, `create-image` operates in a two-pass fashion.

To begin with, all tags are created and lumped together. This is done first by calling `finalize()` on each tag, followed by writing them all out to disk. The `finalize()` call allows tags to update internal fields, but does not allow tags to change their size at all.

The `finalize()` argument takes a single argument, which describes how many additional bytes have been added to the args structure. This function may do nothing, in which case it returns zero. It may also request that additional data be added to the output, in which case it should return a nonzero value.

After all tags have been written, the `last_data()` function is called for each tag. This allows the tag to append data to the end of the structure.

An example of tags that do this include the kernel and the initial programs. They return the size of their payload in the `finalize()` call. They also use this call to update the tag data prior to writing it out.

They also write the actual contents of the tag during the `last_data()` call.