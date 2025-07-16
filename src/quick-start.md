# Quick Start - Hello Embedded World !

Before diving into the theory and concepts of how everything works, let's jump straight into action. Use this simple code to turn on the onboard LED of the ESP32 DevKit.

### Setup project

To start the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 blinky
```

This will open a screen asking you to select options. For now, we dont need to select any options. Just save it by pressing "s" in the keyboard.

Next, navigate to the project folder:
```sh
cd blinky
```


## The full code

Open the `src/bin/main.rs` file. You will find a simple "Hello, World" program inside. We are going to replace it with a different kind of "Hello, World" for the embedded world by making an LED on the board blink. Just copy and paste the code below into the main.rs file.

Do not worry about the code for now. We will explain everything in the next chapter. For now, we just want to see something exciting happen!

```rust
#![no_std]
#![no_main]

use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::main;
use esp_hal::time::{Duration, Instant};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[main]
fn main() -> ! {
    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let mut led = Output::new(peripherals.GPIO2, Level::High, OutputConfig::default());

    loop {
        led.toggle();
        blocking_delay(Duration::from_millis(500));
    }
}

fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```

## Flash - `Run Rust Run`
All that's left is to flash the code onto our device and watch it go! The onboard LED should start blinking.

Run the following command from your project folder:
```rust
cargo run
```

To run in release mode
```rust
cargo run --release
```
