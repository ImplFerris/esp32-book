# Write Rust Code for Motion Detection Using a PIR Sensor and ESP32

Let's write a simple program that prints a message whenever motion is detected. This will help us fine-tune the PIR sensor settings and grasp some basic concepts. Once that's done, we'll build a complete burglar alarm simulation with a buzzer and an onboard LED (or an external LED, which you can adjust as needed) to make it more exciting.


### Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 pir-sensor
```

This will bring up a screen asking you to select options.

In this exercise, we will be printing messages to the system console. In order to do that, we will enable the logging feature.

So, scroll to "Flashing, logging and debugging (espflash)" and hit Enter.

Select "Use defmt to print messages".

Just save it by pressing "s" on the keyboard.


## Sensor Output Pin to ESP32's Input

We'll configure GPIO 33 as an input pin with an initial pull-down state. This pin is connected to the PIR sensor's output, which goes HIGH whenever motion is detected.

```rust
let sensor_pin = Input::new(
    peripherals.GPIO33,
    InputConfig::default().with_pull(Pull::Down),
);
```

## The logic
The idea is simple: we continuously check the sensor's output in a loop. When the sensor's output goes HIGH, we print the message "Motion detected" and add a short delay.

```rust
loop {
    if sensor_pin.is_high() {
        info!("Motion detected");
        blocking_delay(Duration::from_millis(100));
    }
    blocking_delay(Duration::from_millis(100));
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

use defmt::info;
use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Input, InputConfig, Pull};
use esp_hal::main;
use esp_hal::time::{Duration, Instant};
use esp_println as _;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[main]
fn main() -> ! {
    // generator version: 0.3.1

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let sensor_pin = Input::new(
        peripherals.GPIO33,
        InputConfig::default().with_pull(Pull::Down),
    );

    loop {
        if sensor_pin.is_high() {
            info!("Motion detected");
            blocking_delay(Duration::from_millis(100));
        }
        blocking_delay(Duration::from_millis(100));
    }
}

fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```
