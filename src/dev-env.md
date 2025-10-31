# Development Environment

The [official docs](https://docs.espressif.com/projects/rust/book/getting-started/index.html) provides more comprehensive setup instructions. However, I will quickly cover the essential tools and setup needed for our exercises. If you encounter any issues, refer to the official documentation for troubleshooting.

## cargo-binstall

This is to install Rust binaries without building from source using cargo install or manually downloading packages, you can use cargo-binstall. We'll use this tool to install the espflash tool next.

```rust
cargo install cargo-binstall
```

## espflash
"espflash is a serial flasher utility, based on esptool.py, for Espressif SoCs and modules."  This will be the tool used (when we are not using probe-rs) to put our code into the device and run it. 

```sh
cargo binstall espflash
```

If you encounter any problems, you can try installing the exact version used when this book was written.

```sh
cargo binstall espflash@4.2.0
```

After installation, type the espflash command to verify that it works.

```sh
espflash --version
```

## Template by ESP-RS

We will be using the templates provided by [ESP-RS](https://docs.espressif.com/projects/rust/book/getting-started/tooling/esp-generate.html), which offer two sets:  

- **esp-generate**: A `no_std` template. This is the one we will focus on most of the time.  
- **esp-idf-template**: A `std` template.

## esp-generate
The **esp-generate** tool is used for creating `no_std` applications. Currently, it supports the ESP32, ESP32-C2/C3/C6, ESP32-H2, and ESP32-S2/S3. 

```sh
cargo install esp-generate --locked
```

If you want to follow the code exactly as it is in this project, install this esp-generate version used for generating the examples:
```sh
cargo install esp-generate@1.0.0 --locked
```


### Creating project with `esp-generate`

For this book, we will be using the ESP32. I highly recommend using the same hardware to make it easier to follow along.

```sh
# Replace PROJECT_NAME and --chip with your specific chip and project name.
esp-generate --chip esp32 PROJECT_NAME
```


## Toolchains for RISC-V and Xtensa Targets

You will also need `espup` to install the necessary toolchains. You can find details [here](https://docs.espressif.com/projects/rust/book/getting-started/toolchain.html?highlight=espup#xtensa-devices).

```sh
cargo binstall espup
```

or exact version used at the time of writing this book:
```sh
cargo binstall espup@0.16.0
```

Then install the toolchain
```sh
espup install
```

If you want to use the project I created as it is, you might need the exact Rust toolchain version. You can use this command:
```sh
espup install --toolchain-version 1.90.0
```

**NOTE:** Install this specific version **only** if you plan to clone and run the project examples exactly as they are. Using a mismatched version may lead to weird errors (example error: asm! macro is not allowed in naked functions)


## Using the Project example Without Modifications

When you create a project with the esp-generate, it automatically sets "esp" as the toolchain channel. If you want to "clone" and use existing projects instead of creating one from scratch, you need to specify the toolchain name as "book-1.0.0" (as the project's rust-toolchain.toml configured with toolchain name book-1.0.0). This applies only if you're cloning a project from the esp32-projects repository and want to run it without any modifications.

```sh
espup install --name book-1.0.0 --toolchain-version 1.90.0
```
