# Linker Script

Did you try to build the project again? Yup, it no longer compiles. So, what happened?

The problem is: the linker does not know how to lay out your program in memory. Embedded systems like the ESP32 don't have an operating system to handle memory, so we must tell the linker exactly where code, data, and stack should go.

To fix this, we provide a linker script. This is a simple text file that tells the linker things like:

- Where to place code in memory

- Where the stack starts

- How big each memory region is

The linker script we will use is called linkall.x. This script is provided by the [esp-hal](https://github.com/esp-rs/esp-hal/tree/8c64b09ba77d6ecb52eb73e7cea43d37a90d0dab/esp-hal/ld/esp32). It contains memory layout settings suitable for ESP32.

## Cargo config

We can update the ".cargo/config.toml" file with the following lines to fix the issue. However, we will not use this approach. Instead, we will use the build.rs file.

```toml
[target.xtensa-esp32-none-elf]
rustflags = ["-C", "link-arg=-Tlinkall.x"]
```

> ⚠️ Don't add the linker argument in both .cargo/config.toml and build.rs. It will cause a conflict and result in a build error. Use only one method.

## Build.rs file

The esp-generate tool adds this line to the build.rs file. It also does a few more things along with it, which we are not covering here. To match that structure, we will also add the linker argument using a build.rs file.


Now create a new file named build.rs in the root of your project directory (not inside the src/ folder), and add the following code:
 
```rust
fn main() {
    // make sure linkall.x is the last linker script (otherwise might cause problems with flip-link)
    println!("cargo:rustc-link-arg=-Tlinkall.x");
}
```

With this, you should now be able to compile your project:

```sh
cargo build
```
