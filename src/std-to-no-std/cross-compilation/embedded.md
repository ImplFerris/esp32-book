# Compiling for Microcontroller

Now let's talk about embedded systems. When it comes to compiling Rust code for a microcontroller, things work a little differently from normal desktop systems. Microcontrollers don’t usually run a full operating system like Linux or Windows. Instead, they run in a minimal environment, often with no OS at all. This is called a bare-metal environment.

Rust supports this kind of setup through its **no_std** mode. In normal Rust programs, the standard library (`std`) handles things like file systems, threads, heap allocation, and I/O. But none of those exist on a bare-metal microcontroller. So instead of std, we use a much smaller `core` library, which provides only the essential building blocks.

## ESP32 and `std`

Now here's something interesting: the ESP32 actually can run with `std`, but only if you use the official ESP-IDF framework. ESP-IDF provides OS-like features. That makes it possible to use std-based Rust code on the ESP32. 

In this book, we will not be using `std`. We'll focus entirely on the bare-metal approach using `no_std`.

## The Target Triple for ESP32

To compile Rust code for a microcontroller, we need to specify a custom target. For bare-metal ESP32 development, the target triple looks like this:

```sh
xtensa-esp32-none-elf
```

Let’s break this down:

- **Architecture (xtensa)**: The ESP32 is based on the Xtensa architecture, not ARM or x86.
- **Vendor (esp32)**: Refers to the specific chip family from Espressif.
- **OS (none)**: This tells Rust that we are not using an operating system - it's bare-metal.
- **Format (elf)**: This fourth part is important. In many targets, the fourth entry is an ABI (like `gnu` or `msvc`). But here, it's not an ABI - it's the object file format. It will be ELF (Executable and Linkable Format), commonly used in embedded systems.


Wait a minute... You might be wondering: we never used the "rustup target add" command for ESP32! That's because the espup tool takes care of it. We already set it up in the development environment section. In fact, you can't directly add the ESP32 target using the usual rustup target command. Instead, espup installs a separate Rust toolchain under the name "esp" (or any custom name you choose), and that toolchain already comes with ESP32 target support built in.


## Cargo Config

In the quickstart, you might have noticed that we never manually passed the --target flag when running the cargo command. So how did it know which target to build for? That's because the target was already configured in the .cargo/config.toml file.

This file lets you store cargo-related settings, including which target to use by default. To set it up for ESP32, create a .cargo folder in your project root and add a config.toml file with the following content:

```toml
[build]
target = "xtensa-esp32-none-elf"
```

Now you don't have to pass --target every time. Cargo will use this automatically.
