# no_std

Rust has two main foundational crates: std and core.

- The std crate is the standard library. It gives you things like heap allocation, file system access, threads, and println!.

- The core crate is a minimal subset. It contains only the most essential Rust features, like basic types (Option, Result, etc.), traits, and few other operations. It doesn't depend on an operating system or runtime.


When you try to build the project at this stage, you'll get a bunch of errors. Here's what it looks like:

```sh
error[E0463]: can't find crate for `std`
  |
  = note: the `xtensa-esp32-none-elf` target may not support the standard library
  = note: `std` is required by `std_to_no_std` because it does not declare `#![no_std]`
  = help: consider building the standard library from source with `cargo build -Zbuild-std`

error: cannot find macro `println` in this scope
 --> src/main.rs:2:5
  |
2 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

For more information about this error, try `rustc --explain E0463`.
```

There are so many errors here. Lets fix one by one. The first error says the target may not support the standard library. That's true. We already know that. The problem is, we didn't tell Rust that we don't want to use std.  That's where no_std attribute comes into play.


## #![no_std]

The #![no_std] attribute disables the use of the standard library (std). This is necessary most of the times for embedded systems development, where the environment typically lacks many of the resources (like an operating system, file system, or heap allocation) that the standard library assumes are available.

In the top of your src/main.rs file, add this line:

```rs
#![no_std]
```

That's it. Now Rust knows that this project will only use the core library, not std.

## Println

The println! macro comes from the [std crate](https://doc.rust-lang.org/std/macro.println.html). Since we're not using std in our project, we can't use println!. Let's go ahead and remove it from the code.

Now the code should be like this

```rust
#![no_std]
fn main() {
    
}
```

With this fix, we've taken care of two errors and cut down the list. There's still one more issue showing up, and we'll fix that in the next section.


**Resources:**
- [Rust official doc](https://doc.rust-lang.org/reference/names/preludes.html#the-no_std-attribute)
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/intro/no-std.html)
- [Writing an OS in Rust Book](https://os.phil-opp.com/freestanding-rust-binary/#the-no-std-attribute)
