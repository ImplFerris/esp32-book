# Toolchain

Once you add the target in the config.toml file and try to build the project using cargo build, it will no longer compile. At this point, you'll see an error like this:

```sh
'esp32' is not a recognized processor for this target (ignoring processor)
'esp32' is not a recognized processor for this target (ignoring processor)
error[E0463]: can't find crate for `std`
  |
  = note: the `xtensa-esp32-none-elf` target may not be installed
  = help: consider downloading the target with `rustup target add xtensa-esp32-none-elf`

For more information about this error, try `rustc --explain E0463`.
error: could not compile `std_to_no_std` (bin "std_to_no_std") due to 1 previous error
```

The message says esp32 is not a recognized processor and also complains that the target might not be installed. But hang on, didn't we already install everything using the espup tool? Yep, we did.

So what happened? By default, Rust uses your system's global toolchain. That one doesn't know anything about ESP32. espup installed a separate toolchain called "esp" (you can give it a different name too). We just need to tell Rust to use it.

In your project root, create a file called rust-toolchain.toml and add this:

```toml
[toolchain]
channel = "esp"
```

You can use a custom toolchain instead of the default "esp" toolchain. In this book, I'm using a custom toolchain called "book-1.0.0". Just make sure your toolchain is properly installed and that you use the exact name; otherwise, you'll see an error saying it's not installed.

## Build-std

Try to build again. It still won't compile, but this time the error looks a little different. Now it just says it can't find the crate for std.

```sh
error[E0463]: can't find crate for `std`
  |
  = note: the `xtensa-esp32-none-elf` target may not be installed
  = help: consider downloading the target with `rustup target add xtensa-esp32-none-elf`
  = help: consider building the standard library from source with `cargo build -Zbuild-std`

For more information about this error, try `rustc --explain E0463`.
error: could not compile `std_to_no_std` (bin "std_to_no_std") due to 1 previous error
```

The thing is, the ESP32 target is a [Tier 3 target](https://doc.rust-lang.org/beta/rustc/target-tier-policy.html#tier-3-target-policy). That means Rust doesn't ship precompiled standard libraries for it. We have to build the core library from source ourselves.

To do that, update the .cargo/config.toml file with this:
```toml
[unstable]
build-std = ["core"]
```

Try building again. It still won't compile. But don't worry, we're getting closer. In the next section, we'll fix the next set of errors.
