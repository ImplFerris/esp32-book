# Flash

Nope, we are not talking about the fastest man alive. We are talking about writing our binary into the microcontroller and running it.

To do that, we will use a helpful tool called espflash. There are other tools like probe-rs, but we will stick to espflash for now.


You can flash and monitor your program with the following command:

```rust
espflash flash  --monitor --chip esp32 ./target/xtensa-esp32-none-elf/debug/std_to_no_std
```

## Cargo run

Typing this long command every time can get annoying. So let's make it easier by updating the ".cargo/config.toml" file. We can tell Cargo to use espflash automatically when we run cargo run.

Add this section to your .cargo/config.toml:

```toml
[target.xtensa-esp32-none-elf]
runner = "espflash flash --monitor --chip esp32"
```

Now, you can just type:

```sh
cargo run
```
and your program will be flashed and run on the ESP32.


Phew... We started with a standard library binary and converted it into a no_std binary for the ESP32. Finally, we can now see the LED blinking.

Good news - we will not have to go through this setup every time we create a new project. That is exactly why tools like esp-generate (or cargo-generate) are super helpful in embedded programming. They set up all the tricky parts so we can jump straight into writing code.
