# Blinking an LED with ESP RTOS (Embassy) on ESP32 in Rust

Let's re-setup the blinky project but with embassy support.

### Setup project

To start the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 blinky-embassy
```

This time, we need to select "Add Embassy framework support". Since this requires unstable feature, we'll first click "Enable unstable HAL features" and then proceed to select Embassy support. Finally, we'll press 's' to save the generated project with Embassy support.

## Key points

If you notice, the main function is now marked as async, along with a few other changes in the code. However, the core logic for blinking the LED remains the same.

The key addition is this part:

```rust
let timg0 = TimerGroup::new(peripherals.TIMG0);
esp_rtos::start(timg0.timer0);
```
This sets up a timer that Embassy needs to handle async tasks like delays. We create a timer group using hardware timer TIMG0, then pass one of its timers to esp_rtos::start to let Embassy use it for time-based operations.


## The Full code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::timer::timg::TimerGroup;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[esp_rtos::main]
async fn main(_spawner: Spawner) -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0);

    let mut led = Output::new(peripherals.GPIO2, Level::High, OutputConfig::default());

    loop {
        led.toggle();
        Timer::after(Duration::from_secs(1)).await;
    }
}
```

 
## Clone the existing project

You can clone (or refer) project I created and navigate to the `async-projects/blinky-embassy` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/async-projects/blinky-embassy
```
