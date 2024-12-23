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



