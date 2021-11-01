# Xous Build System Overview

The Xous build system uses the `xtask` concept to perform complex tasks without needing an external build system.

The `xtask` concept is simply an alias under `.cargo/config` that turns `cargo xtask` into `cargo run --package xtask --`.

Therefore, all complex operations from building the kernel to constructing an output image are handled by `xtask/src/main.rs`, which is compiled and run as a normal Rust program.

## Step 0: Build the Build System

When you type `cargo xtask`, the build system will compile `xtask/src/main.rs`. This happens automatically.

## Step 1: Build the Kernel

The build system runs `cargo build --package kernel --release --target riscv32imac-unknown-xous-elf` in the `kernel/` directory.

## Step 2: Build the Initial Programs

The build system runs `cargo build --target riscv32imac-unknown-xous-elf` with every initial program appended as a `--package` argument.

## Step 3: Build the Loader

The build system runs `cargo build --target riscv32imac-unknown-xous-elf --package loader` in the `loader/` directory.

## Step 4: Package it all Up

The build system runs `cargo run --package tools --bin create-image --` followed by arguments to create the mage.