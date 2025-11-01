## Creating a Beep Sound with an Active Buzzer Using ESP32 and Rust

Since you already know that an active buzzer is simple to use, you can make it beep just by powering it. In this exercise, we'll make it beep with just a little code.


### Hardware Requirements
- **Active Buzzer**
- **Female-to-Male** or **Male-to-Male** jumper wires (depending on your setup)

### Generate project using esp-generate

You have done this step already in the quick start section. 

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 active-buzzer
```

This will open a screen asking you to select options. For now, we dont need to select any options. Just save it by pressing "s" in the keyboard.

## Code

We will set GPIO 33 as our output pin with an initial Low state. This is the pin where we connected the positive pin of the buzzer.

```rust
let mut buzzer = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());
```

The logic is straightforward: set the buzzer pin to High for 500 milliseconds, then to Low for another 500 milliseconds in a loop. This causes the buzzer to produce a beeping sound.

```rust
   loop {
        buzzer.set_high();
        blocking_delay(Duration::from_millis(500));
        buzzer.set_low();
        blocking_delay(Duration::from_millis(500));
    }
```

## Helper function for delay
It waits until the specified duration has passed.

```rust
fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```

## Clone the existing project
You can clone (or refer) project I created and navigate to the `active-buzzer` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/active-buzzer
```


## The Full code 
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
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let mut buzzer = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());

    loop {
        buzzer.set_high();
        blocking_delay(Duration::from_millis(500));
        buzzer.set_low();
        blocking_delay(Duration::from_millis(500));
    }
}

fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```
