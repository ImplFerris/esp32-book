# Quick Start

Before diving into the theory and concepts of how everything works, let's jump straight into action. Use this simple code to turn on the onboard LED of the ESP32 DevKit.

## Blink LED with ESP HAL
[ESP HAL](https://github.com/esp-rs/esp-hal) is "no_std Hardware Abstraction Layers for ESP32 microcontrollers"

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

Open the src/bin/main.rs file. It will contain a simple "Hello, World" code. We will modify this code to blink an LED on the board.

## Blinking Code

This code creates a blinking effect by toggling an LED connected to a GPIO pin between high and low states.

#### Import Required Module
Additional import we need to set the LED as output pin
```rust
use esp_hal::gpio::{Io, Level, Output};
```

#### Initialize ESP HAL and Delay
Set up the ESP HAL and configure a delay
```rust
let peripherals = esp_hal::init(esp_hal::Config::default());
let delay = Delay::new();
```

Then lets set the LED GPIO pin "GPIO2" as an output pin with an initial state "High"(LED is turned on):
```rust
let io = Io::new(peripherals.GPIO, peripherals.IO_MUX);
let mut led = Output::new(io.pins.gpio2, Level::High);
```

#### Blinking Loop
Create a loop to toggle the LED state(between High and Low).
```rust
loop {
    led.toggle();
    delay.delay_millis(500);
    led.toggle();
    delay.delay_millis(500);
}
```

## The full code
```rust
#![no_std]
#![no_main]

use esp_hal::delay::Delay;
use esp_hal::prelude::*;
use {esp_backtrace as _};

use esp_hal::gpio::{Io, Level, Output};
#[entry]
fn main() -> ! {
    #[allow(unused)]
    let peripherals = esp_hal::init(esp_hal::Config::default());
    let delay = Delay::new();

    let io = Io::new(peripherals.GPIO, peripherals.IO_MUX);
    let mut led = Output::new(io.pins.gpio2, Level::High);

    loop {
        led.toggle();
        delay.delay_millis(500);
        led.toggle();
        delay.delay_millis(500);
    }
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
