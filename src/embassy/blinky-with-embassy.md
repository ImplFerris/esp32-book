# Blinking an LED with Embassy on ESP32 in Rust

Let's re-setup the blinky project but with embassy support.

### Setup project

To start the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 blinky-embassy
```

This will open a screen prompting you to select options. This time, choose "Adds embassy framework support" and save to generate the project template with Embassy support. However, if you find that the "embassy" option is unavailable, first enable "Enable unstable HAL features", then proceed with selecting "embassy" support.

If you notice, the main function is now marked as async, along with a few other changes in the code. However, the core logic for blinking the LED remains the same.

The key addition is this part:

```rust
let timer0 = TimerGroup::new(peripherals.TIMG1);
esp_hal_embassy::init(timer0.timer0);
```
This sets up a timer that Embassy needs to handle async tasks like delays. We create a timer group using hardware timer TIMG1, then pass one of its timers to esp_hal_embassy::init() to let Embassy use it for time-based operations.


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

 