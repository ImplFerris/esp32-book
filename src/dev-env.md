# Development Environment

The [official docs](https://docs.esp-rs.org/book/installation/index.html) provides more comprehensive setup instructions. However, I will quickly cover the essential tools and setup needed for our exercises. If you encounter any issues, refer to the official documentation for troubleshooting.

## cargo-binstall

This is to install Rust binaries without building from source using cargo install or manually downloading packages, you can use cargo-binstall. We'll use this tool to install the espflash tool next.

```rust
cargo install cargo-binstall
```

## espflash
"espflash is a serial flasher utility, based on esptool.py, for Espressif SoCs and modules."  This will be the tool used (when we are not using probe-rs) to put our code into the device and run it. 

```rust
cargo binstall espflash
```

After installation, type the espflash command to verify that it works.


## Toolchains for RISC-V and Xtensa Targets

You will also need `espup` to install the necessary toolchains. You can find details [here](https://docs.esp-rs.org/book/installation/riscv-and-xtensa.html).

```sh
cargo install espup
espup install
```

If you want to use the project I created as it is, you might need the exact Rust toolchain version. You can use this command:
```rust
espup install --toolchain-version 1.82.0
```

However, I recommend reading the book, understand the concepts and code, and apply them to a newer version. You might face challenges, but you'll learn and even contribute back to this book 😉.

**NOTE:** Install this specific version **only** if you plan to clone and run the project examples exactly as they are. Otherwise, install the latest version.  Using a mismatched version may lead to weird errors (eg: error: asm! macro is not allowed in naked functions)

