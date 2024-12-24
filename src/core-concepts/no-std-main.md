# No Standard and Main in Rust

If you haven't read "The Embedded Rust Book" yet, I highly recommend you to check it out.
[https://docs.rust-embedded.org/book/intro/index.html](https://docs.rust-embedded.org/book/intro/index.html)

## #![no_std]
The `#![no_std]` attribute disables the use of the standard library (std). This is necessary most of the times for embedded systems development, where the environment typically lacks many of the resources (like an operating system, file system, or heap allocation) that the standard library assumes are available.

**Related Resources:**
- [Rust official doc](https://doc.rust-lang.org/reference/names/preludes.html#the-no_std-attribute)
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/intro/no-std.html)
- [Writing an OS in Rust Book](https://os.phil-opp.com/freestanding-rust-binary/#the-no-std-attribute)

## #![no_main]
The `#![no_main]` attribute is to indicate that the program won't use the standard entry point (fn main). Instead, it provides a custom entry point, usually required when working with embedded systems where the runtime environment is minimal or non-existent.

**Related Resources:**
- [Rust official doc](https://doc.rust-lang.org/reference/crates-and-source-files.html?highlight=no_main#the-no_main-attribute)
- [Writing an OS in Rust Book](https://os.phil-opp.com/freestanding-rust-binary/#overwriting-the-entry-point)


## Panic Handler
A panic handler is a function in Rust that defines what happens when your program encounters a panic. In environments without the standard library (when using no_std attribute), you need to create this function yourself using the #[panic_handler] attribute. This function must follow a specific format and can only appear once in your program. It provides details about the error, such as where it happened and why. By setting up a panic handler, you can choose how to respond to errors, like logging them for later review or stopping the program completely.

You don't have to define your own panic handler function; you can use existing crates such as panic_halt or panic_probe instead.

For example, we used the panic_halt crate to halt execution when a panic occurs. 
```rust
use panic_halt as _;
```
The program will stop and remain in this infinite loop whenever a panic occurs.

In fact, the [panic_halt crate's code](https://github.com/korken89/panic-halt/blob/master/src/lib.rs) implements a simple panic handler, which looks like this:
```rust
use core::panic::PanicInfo;
use core::sync::atomic::{self, Ordering};

#[inline(never)]
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {
        atomic::compiler_fence(Ordering::SeqCst);
    }
}
```

**Related Resources:**
- [Rust official doc](https://doc.rust-lang.org/nomicon/panic-handler.html)
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/start/panicking.html)
