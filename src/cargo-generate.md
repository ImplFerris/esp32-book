# Project Template with `cargo-generate`

"cargo-generate is a developer tool to help you get up and running quickly with a new Rust project by leveraging a pre-existing git repository as a template."

Read more about [here](https://github.com/cargo-generate/cargo-generate).
 
## Prerequisites

Before starting, ensure you have the following tools installed:

- [Rust](https://www.rust-lang.org/tools/install)
- [cargo-generate](https://github.com/cargo-generate/cargo-generate) for generating the project template.

You can install `cargo-generate` using the following command:
```sh
cargo install cargo-generate
```

## Template by ESP-RS
We will be using the templates provided by [ESP-RS](https://docs.esp-rs.org/book/writing-your-own-application/generate-project/index.html#esp-generate), which offer two sets:  
- **esp-generate**: A `no_std` template. This is the one we will focus on most of the time.  
- **esp-idf-template**: A `std` template.

## esp-generate
The **esp-generate** tool is used for creating `no_std` applications. Currently, it supports the ESP32, ESP32-C2/C3/C6, ESP32-H2, and ESP32-S2/S3. 

```sh
cargo install esp-generate
```

If you want to follow the code exactly as it is in this project, install this esp-generate version used for generating the examples:
```sh
cargo install esp-generate@0.2.1
```
The related rust toolchain for this version is 1.82.0


### Creating project with `esp-generate`
For this book, we will be using the ESP32. I highly recommend using the same hardware to make it easier to follow along.

```sh
# Replace PROJECT_NAME and --chip with your specific chip and project name.
esp-generate --chip esp32 PROJECT_NAME
```
