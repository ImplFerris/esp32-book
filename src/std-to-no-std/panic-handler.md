# Panic Handler

At this point, when you try to build the project, you’ll get this error:

```sh
error: `#[panic_handler]` function required, but not found
```

When a Rust program panics, it is usually handled by a built-in panic handler that comes from the standard library. But in the last step, we added #![no_std], which tells Rust not to use the standard library. So now, there's no panic handler available by default.

In a no_std environment, you are expected to define your own panic behavior, because there's no operating system or runtime to take over when something goes wrong.

We can fix this by adding our own panic handler. Just create a function with the #[panic_handler] attribute. The function must accept a reference to PanicInfo, and its return type must be !, which means the function never returns.

Add this to your src/main.rs:

```rust
#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

## Panic crates

There are some ready-made crates that provide a panic handler function for no_std projects. One simple and commonly used crate is "panic_halt", which just halts the execution when a panic occurs.


```rust
use panic_halt as _;
```
This line pulls in the panic handler from the crate. Now, if a panic happens, the program just stops and stays in an infinite loop.

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

You can either use an external crate like this, or write your own panic handler function manually. It's up to you. The esp-template tool actually writes the panic handler directly into the main.rs file instead of pulling in a separate crate.


**Resources:**
- [Rust official doc](https://doc.rust-lang.org/nomicon/panic-handler.html)
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/start/panicking.html)
