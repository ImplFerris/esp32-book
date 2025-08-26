# LED

Next step is to configure the onboard LED of the DevKit v1. How are we going to do that? For that, we need to control a GPIO pin.

## GPIO 
GPIO stands for General Purpose Input/Output. These are pins on the microcontroller that can either read digital signals (input) or send digital signals (output).

- Want to read a button? Use a GPIO as input.

- Want to turn on an LED? Use a GPIO as output.

- Want to talk to a sensor or control a motor? GPIO pins are the starting point.

GPIOs are the most basic and most essential way your program talks to the outside world.

## What pin is the onboard LED connected to?

If you are using a bare ESP32 chip, there's no onboard LED.

But, we are using the ESP32 DevKit v1 development board. On this board, there is onboard LED which is connected to GPIO2. So that's the pin we'll use next.

## Initialize LED 

To control the onboard LED, we configure GPIO2 as an output pin. 

Adding this necessary import:

```rust
use esp_hal::gpio::{Level, Output, OutputConfig};
```

Add the following line inside the main function:

```rust
let mut led = Output::new(peripherals.GPIO2, Level::High, OutputConfig::default());
```

The peripherals.GPIO2 part gives us access to that GPIO2 pin. We set its initial state to Level::High, which turns the LED on. The OutputConfig::default() uses standard settings to make it an output pin.  Note here, we marked the led variable as mutable because we will change its state later.  We can use the toggle() function to switch the LED state between high and low, allowing us to blink the LED.

## The Main Loop

The main loop continuously toggles the LED state using a simple approach. We call the toggle() method on our LED output pin, which switches it between high and low states. Between each toggle, we introduce a 500-millisecond delay to create a visible blinking effect.

```rust
loop {
    led.toggle();
    blocking_delay(Duration::from_millis(500));
}
```

The blocking_delay function provides a simple timing mechanism by busy-waiting until the specified duration has elapsed. It captures the current time and continuously checks if the time has passed:

```rust
fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```

## Final code

Here's the full code:

```rust
#![no_std]
#![no_main]

use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::main;
use esp_hal::time::{Duration, Instant};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

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
