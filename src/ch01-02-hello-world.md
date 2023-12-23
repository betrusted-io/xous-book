# Hello, World!

* You will need the latest stable Rust. For now, Xous is tightly coupled to the latest stable Rust toolchain.
* One should be able to run `cargo xtask run` after cloning [xous-core](https://github.com/betrusted-io/xous-core), and it will pop up a "host-mode" emulated version of Precursor.
  - You should be prompted to install the `xous` target the very first time you run `cargo`.
     - If you have updated `rust` or have tinkered with `std` on your system, you can re-install the `xous` target with `cargo xtask install-toolkit --force`, and then run `rm -r target` to force remove stale build files.
  - You may also need to install some additional libraries, such as `libxkbdcommon-dev`.
  - :warning: hosted mode is literally Xous running on your local host, which means it supports more features than Xous on native hardware:
     - We do not have `tokio` support planned anytime soon.
     - We do not have `File` support in Xous; instead, we have the `pddb`.
     - `Net` support in actively in development and we hope to have fairly robust support for `libstd` `Net` but, note that `socket2` crate (which is not part of Rust `libstd`) does *not* recognize Xous as a supported host.
  - It is recommended to try compiling your configuration for a real hardware target or Renode early on to confirm compatibility, before doing extensive development in hosted mode.
* Make your own app:
  - Please refer to the [manifest.json](https://github.com/betrusted-io/xous-core/blob/main/apps/README.md) documentation for integration notes
  - [`repl` app demo](https://github.com/betrusted-io/xous-core/blob/main/apps/repl/README.md) is the starting point for users who want to interact with their device by typing commands. This demo leverages more of the Xous UX built-in frameworks.
  - [`ball` app demo](https://github.com/betrusted-io/xous-core/blob/main/apps/ball/README.md) is the starting point for users who prefer to run close to the bare iron, getting only key events and a framebuffer for crafting games and bespoke apps.
