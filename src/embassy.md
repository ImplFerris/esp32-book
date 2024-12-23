## Async

Async programming is a type of concurrent programming that allows tasks to run concurrently without blocking each other.  In embedded systems, it enables microcontrollers to handle multiple tasks, such as reading sensors or controlling other peripherals without waiting for each task to finish. You can read the "[Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/intro.html)" for more details.

## Embassy

[Embassy](https://github.com/embassy-rs/embassy) is a powerful framework for building safe, efficient, and asynchronous embedded applications in Rust. You can use it with ESP32, Pico and other microcontrollers. 

Let's re-setup the same blinky project but with embassy support.

### Setup project

To start the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 blinky-embassy
```

This will open a screen asking you to select options. But, this time we select "Adds `embassy` framework support" and save it to generate the project template with support of embassy.

If you notice, the main function is now marked as async, along with a few other changes in the code. However, the core logic for blinking the LED remains the same.

## The Full code

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::Timer;
use esp_backtrace as _;
use esp_hal::{
    gpio::{Io, Level, Output},
    prelude::*,
};
use log::info;

#[main]
async fn main(_spawner: Spawner) {
    let peripherals = esp_hal::init({
        let mut config = esp_hal::Config::default();
        config.cpu_clock = CpuClock::max();
        config
    });

    esp_println::logger::init_logger_from_env();

    let timg0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG0);
    esp_hal_embassy::init(timg0.timer0);
    info!("Embassy initialized!");

    let io = Io::new(peripherals.GPIO, peripherals.IO_MUX);
    let mut led = Output::new(io.pins.gpio2, Level::High);
    loop {
        led.set_high();
        Timer::after_millis(500).await;

        led.set_low();
        Timer::after_millis(500).await;
    }
}
```


## Reference

- [The Embedded Rust Book](https://docs.rust-embedded.org/book/intro/index.html)
