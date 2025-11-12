# Quick Start - Hello Embedded World !

Before diving into the theory and concepts of how everything works, let's jump straight into action. Use this simple code to turn on the onboard LED of the ESP32 DevKit.

### Setup project

To start the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 esp32-quick
```

This will open a screen asking you to select options. For now, we dont need to select any options. Just save it by pressing "s" in the keyboard.

> You can also choose the editor integration option and then select the specific editor you are using.

Next, navigate to the project folder:
```sh
cd esp32-quick
```

## The full code

Open the `src/bin/main.rs` file. You will find a simple "Hello, World" program inside. We are going to replace it with a different kind of "Hello, World" for the embedded world by making an LED on the board blink. Just copy and paste the code below into the main.rs file.

Do not worry about the code for now. We will explain everything in the next chapter. For now, we just want to see something exciting happen!

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::main;
use esp_hal::time::{Duration, Instant};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

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

## Clone the existing project
You can clone (or refer) project I created, if you encounter any issue.

```sh
git clone https://github.com/ImplFerris/esp32-quick
cd esp32-quick/
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
