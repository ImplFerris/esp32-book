# Write Rust Code to Control ESP32 LED via Wi-Fi

You can configure a web server on the ESP32 to receive instructions via web requests, allowing you to control connected devices or execute specific actions. For instance, you can use a browser on your computer or mobile phone to send commands to turn an LED on or off, adjust motor speed, or retrieve sensor data.

In this section, we will create a simple web page that allows us to turn an LED on or off.

## Project base
This time, we are not going to set up the project with esp-generate. Instead, we will copy the webserver-base project and work on top of that. 

I recommend you to read these section before you proceed furhter; This will avoid unnecessary repetition of code and explanations.
- [Creating Web Server](../web-server/index.md) 
- [Assigning Static IP](../static-ip.md)
 

```sh
git clone https://github.com/ImplFerris/esp32-projects
cp -r esp32-projects/webserver-base ~/YOUR_PROJECT_FOLDER/wifi-led
```

**Project Structure**:

```
├── build.rs
├── Cargo.toml
├── rust-toolchain.toml
├── src
│   ├── bin
│   │   └── async_main.rs
│   ├── index.html
│   ├── led.rs
│   ├── lib.rs
│   ├── web.rs
│   └── wifi.rs
```

## Serde 
Serde is a Rust crate used for serializing and deserializing data structures. We will use it to handle the JSON data exchanged between the backend and the frontend.

Update the Cargo.toml with the following:
```toml
serde = { version = "1.0.217", default-features = false, features = ["derive"] }
```

## LED Task 

I placed these code in the "led.rs" module. 

First, we'll create an Embassy task to toggle the onboard LED state based on the value stored in the LED_STATE variable, which will use the AtomicBool type. "Atomic types provide primitive shared-memory communication between threads, and are the building blocks of other concurrent types".  To learn more about Atomic types, refer to the Rust standard library documentation on [atomics](https://doc.rust-lang.org/beta/core/sync/atomic/index.html) or the [Rust Atomics and Locks](https://marabos.nl/atomics/) book.

```rust
use core::sync::atomic::{AtomicBool, Ordering};

use embassy_time::{Duration, Timer};
use esp_hal::gpio::Output;

pub static LED_STATE: AtomicBool = AtomicBool::new(false);

#[embassy_executor::task]
pub async fn led_task(mut led: Output<'static>) {
    loop {
        if LED_STATE.load(Ordering::Relaxed) {
            led.set_high();
        } else {
            led.set_low();
        }
        Timer::after(Duration::from_millis(50)).await;
    }
}
```

In the led_task function, we take an LED pin as argument and continuously checks the value of the LED_STATE variable in a loop. We read the value using the load method with Ordering::Relaxed. If the value is true, we turn on the LED. Otherwise, we turn off the LED.

In the main function, we spawn the led_task to run it in the background. We will pass the GPIO 2 pin(If you want to use an external LED, replace it with the pin to which you connected the LED), which is the onboard LED, and we will set the initial state of the LED to Low.

```rust
// LED Task
spawner.must_spawn(lib::led::led_task(Output::new(
    peripherals.GPIO2,
    Level::Low,
)));
```
















