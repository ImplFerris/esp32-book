# no_main

When you try to build at this stage, you'll get an error saying the main function requires the standard library. What?! (I controlled my temptation to insert a Mr. Bean meme here since not everyone will like meme.) So what now? Where does the program even start?

In embedded systems, we don't use the regular "fn main" that relies on the standard library. Instead, we have to tell Rust that we'll bring our own entry point. And for that, we use the no_main attribute.

The `#![no_main]` attribute is to indicate that the program won't use the standard entry point (fn main).

In the top of your src/main.rs file, add this line:

```rs
#![no_main]
```

## Declaring the Entry Point

Now that we've opted out of the default entry point, we have to tell Rust what function to start with. The esp-hal crate provides a handy attribute for this: #[main].

This attribute marks your function as the custom entry point. The function you tag with #[main] will be called by the reset handler after RAM is initialized.

First, update your Cargo.toml to include the esp-hal crate. Since it supports multiple chips, we need to enable the one we're using (in our case, esp32) using a feature flag.

```toml
[dependencies]
esp-hal = { version = "=1.0.0-beta.1", features = ["esp32"] }
```

Then, in your main.rs, set up the entry point like this:

```rust
use esp_hal::main;

#[main]
fn main() {}

```

But wait, one more step is left.

The entry function must have a special signature - it should never return. That means the return type must be ! (never type). This tells Rust that your function will keep running forever, which is common in embedded systems.

So let's update the main function like this:

```rust
#[main]
fn main() -> ! {
    loop {}
}
```

## Are we there yet?

Hoorah! Now try building the project - it should compile successfully.

You can inspect the generated binary using the file command:

```sh
file target/xtensa-esp32-none-elf/debug/std_to_no_std
```

It will show something like this:

```sh
target/xtensa-esp32-none-elf/debug/std_to_no_std: ELF 32-bit LSB executable, Tensilica Xtensa, version 1 (SYSV), statically linked, with debug_info, not stripped
```
As you can see, the binary is built for a 32-bit Xtensa target. That means our base setup for ESP32 is working.

But are we there yet? Not quite. We've crossed half the stage - we now have a valid binary ready for ESP32, but there's more to do before we can run it on real hardware.


**Resources:**
- [Rust official doc](https://doc.rust-lang.org/reference/crates-and-source-files.html?highlight=no_main#the-no_main-attribute)
- [Writing an OS in Rust Book](https://os.phil-opp.com/freestanding-rust-binary/#overwriting-the-entry-point)

