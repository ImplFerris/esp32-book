# Write Rust Code for Motion Detection Using a PIR Sensor and ESP32

Let's write a simple program that prints a message whenever motion is detected. This will help us fine-tune the PIR sensor settings and grasp some basic concepts. Once that's done, we'll build a complete burglar alarm simulation with a buzzer and an onboard LED (or an external LED, which you can adjust as needed) to make it more exciting.


### Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 pir-sensor
```

This will open a screen asking you to select options. We dont need to select any options. Just save it by pressing "s" in the keyboard.

## Sensor Output Pin to ESP32's Input

We'll configure GPIO 33 as an input pin with an initial pull-down state. This pin is connected to the PIR sensor's output, which goes HIGH whenever motion is detected.

```rust
let sensor_pin = Input::new(peripherals.GPIO33, Pull::Down);
```

## The logic
The idea is simple: we continuously check the sensor's output in a loop. When the sensor's output goes HIGH, we print the message "Motion detected" and add a short delay.

```rust
loop {
    if sensor_pin.is_high() {
        println!("Motion detected");
        delay.delay(100.millis());
    }
    delay.delay(100.millis());
}
```


## Clone the existing project
You can clone (or refer) project I created and navigate to the `pir-sensor` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/pir-sensor
```

## The Full code

```rust
#![no_std]
#![no_main]

use esp_backtrace as _;
use esp_hal::delay::Delay;
use esp_hal::gpio::{Input, Pull};
use esp_hal::prelude::*;
use esp_println::println;

#[entry]
fn main() -> ! {
    let peripherals = esp_hal::init({
        let mut config = esp_hal::Config::default();
        config.cpu_clock = CpuClock::max();
        config
    });

    esp_println::logger::init_logger_from_env();

    let sensor_pin = Input::new(peripherals.GPIO33, Pull::Down);

    let delay = Delay::new();
    loop {
        if sensor_pin.is_high() {
            println!("Motion detected");
            delay.delay(100.millis());
        }
        delay.delay(100.millis());
    }
}
```
