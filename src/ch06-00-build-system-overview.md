# Xous Build System Overview

The Xous build system uses the `xtask` concept to perform complex tasks without needing an external build system.

The `xtask` concept is simply an alias under `.cargo/config` that turns `cargo xtask` into `cargo run --package xtask --`.

Therefore, all complex operations from building the kernel to constructing an output image are handled by `xtask/src/main.rs`, which is compiled and run as a normal Rust program.

## Building Images

Generally, users will want to use `cargo xtask app-image [app 1] [app ..]` to build a Xous image that contains the desired list of applications. The applications are the names of crates contained in the "apps/" directory. Tesulting FLASH images will be called [`loader.bin`](ch05-02-loader.md) and `xous.img` in the `target/riscv32imac-unknown-xous-elf/release` directory.

There are also convenience commands to build emulation images, such as `cargo xtask run` (for hosted mode, where Xous runs directly on your native OS) and `cargo xtask renode-image` (for a Renode image, where a cycle accurate simulation can be run inside the Renode emulator). See Chapter 4 for more information about Renode.

### Build Command Syntax Details

The general format of an `xtask` command is as follows:

```text
cargo xtask [verb] [cratespecs ..]
    [--feature [feature name]]
    [--lkey [loader key]] [--kkey [kernel key]]
    [--app [cratespec]]
    [--service [cratespec]]
    [--no-timestamp]
```

After `xtask`, a `verb` will select one of several pre-configured sets of packages and build targets. Immediately after `verb`, one can specifay a list of 0 or more items that are interpreted as `cratespecs`.

The binary images merged into a FLASH image is usually built from local source, but it can actually come from many locations. Thus each `cratespec` has the following syntax:
- `name`: crate 'name' to be built from local source
- `name@version`: crate 'name' to be fetched from crates.io at the specified version
- `name#URL`: pre-built binary crate of 'name', to be downloadeded from a server at 'URL' after the `#` separator
- `path-to-binary`: file path to a prebuilt binary image on local machine. Files in '.' must be specified as `./file` to avoid confusion with local source

The exact meaning of a `cratespec` depends on the context of the verb. Generally, fully-configured builds interpret the `cratespec` as an `app`, and debug builds interpcet `cratepsec` as a `service`.

Both an `app` and a `service` are Xous binaries that are copied into the final disk image; however, there is an additional step that gets run in the build system for an `app` that looks up its description in `apps/manifest.json` and attempts to configure the launch menu for the app prior to running the build.

Additional crates can be merged in with explicit app/service treatment by preceeding the crate name with either an `--app` flag or `---service` flag.

#### Example: Building a Precursor User Image

If one were building a user image for Precursor hardware, one could use the following command to build a base system that contains no apps.

`cargo xtask app-image`

`app-image` automatically selects a [`utralib`](ch06-03-target-specification.md) hardware target, and populates a set of base services that would be bundled into a user image. Thus this command would create an image with no apps and just the default `shellchat` management interface.

One could add the `vault` app and the `ball` demo app by specifying them as positional arguments like this:

`cargo xtask vault ball`

It is also perfectly fine to specify them using explicit flags like this:

`cargo xtask --app vault --app ball`

#### Example: System Bringup Builds

When doing system bringup, it's often helpful to build just the tiniest subset of Xous, and then merge the service of interest into the disk image. Let's say you are building a tiny service that is located in a separate source tree. Let's say the service is called `test-server` and your workspace is set up like this:

```text
|
|-- xous-core/
|-- test-server/
```

Inside `test-server`, you would have a `src/main.rs` that looks like this:

```rust,noplayground,ignore
use xous_api_log_server as log_server;
use std::{thread, time};

fn main() -> ! {
    log_server::init_wait().unwrap();
    log::set_max_level(log::LevelFilter::Info);
    log::info!("my PID is {}", xous::process::id());

    let timeout = time::Duration::from_millis(1000);
    let mut count = 0;
    loop {
        log::info!("test loop {}", count);
        count += 1;
        thread::sleep(timeout);
    }
}
```

And a `Cargo.toml` that looks like this:
```toml
[package]
name = "test-server"
version = "0.1.0"
edition = "2021"

[dependencies]
xous = "0.9.9"
log = "0.4.14"
xous-api-log-server = {version = "0.1.2", package = "xous-api-log-server"}

```

Inside `test-server`, run this command to build the program:

`cargo build --target riscv32imac-unknown-xous-elf --release`

The command would create a Xous executable in `target/riscv32imac-unknown-elf/release/test-server`.

Then, in the `xous-core` source tree, you can run this command to create a runnable Xous disk image:

`cargo xtask tiny ../test-server/target/riscv32imac-unknown-elf/release/test-server`

The `tiny` verb selects the smallest subset of servers one can have to run the most basic OS functions, and it interprets the path to `test-server` as a `cratespec` which is injected into the `xous.img` file as another service. The Loader will automatically load `test-server` and run it concurrently with all the other Xous services in the `tiny` target.

You could also use the `libstd-test` verb and create the same image, but for a Renode target.

The advantage of this process is that you can iterate rapidly on `test-server` without triggering rebuilds of Xous, since the `test-server` program is entirely out of tree and specified as a binary file path `cratespec` to `xtask`. This is particularly useful during very early hardware bring-up of a new peripheral.

Note that the `tiny` and `libstd-test` targets contain a minimal subset of Xous, and not all `libstd` functions will work; for example, the `Net` and `File` functions would fail because the test image does not contain networking or PDDB services. Generally, for development that relies on higher-level APIs such as networking and filesystem, it's easier to just build the full user image, because beyond a certain point of complexity all the services become inter-dependent upon each other and there is less value in isolating their dependencies.

Binary `cratespec`s are also useful for handling license-incompatible crates. Xous is MIT-or-Apache licensed; thus, one cannot introduce a GPL crate into its source tree. However, one can create an executable from a GPL program, that is then copied into a Xous disk image using binary path `cratespec`s.

## The Internal Flow of the Build System

For those curious as to what the builder does on the inside, here is the general flow of most build operations.

### Step 0: Build the Build System

When you type `cargo xtask`, the build system will compile `xtask/src/main.rs`. This happens automatically.

### Step 1: Build the Kernel

The build system runs `cargo build --package kernel --release --target riscv32imac-unknown-xous-elf` in the `kernel/` directory.

### Step 2: Build the Initial Programs

The build system runs `cargo build --target riscv32imac-unknown-xous-elf` with every initial program appended as a `--package` argument.

### Step 3: Build the Loader

The build system runs `cargo build --target riscv32imac-unknown-xous-elf --package loader` in the `loader/` directory.

### Step 4: Package it all Up

The build system runs `cargo run --package tools --bin create-image --` followed by arguments to create the image.
