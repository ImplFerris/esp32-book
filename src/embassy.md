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

This will open a screen prompting you to select options. This time, choose "Adds embassy framework support" and save to generate the project template with Embassy support. However, if you find that the "embassy" option is unavailable, first enable "Enable unstable HAL features", then proceed with selecting "embassy" support.

If you notice, the main function is now marked as async, along with a few other changes in the code. However, the core logic for blinking the LED remains the same.

## The Full code

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::timer::timg::TimerGroup;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[esp_hal_embassy::main]
async fn main(_spawner: Spawner) {
    // generator version: 0.3.1

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timer0 = TimerGroup::new(peripherals.TIMG1);
    esp_hal_embassy::init(timer0.timer0);

    let mut led = Output::new(peripherals.GPIO2, Level::High, OutputConfig::default());

    loop {
        led.toggle();
        Timer::after(Duration::from_secs(1)).await;
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.0.0-beta.0/examples/src/bin
}
```


## Reference

- [The Embedded Rust Book](https://docs.rust-embedded.org/book/intro/index.html)
