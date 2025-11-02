# Turn on an LED in the Dark with a Photoresistor and ESP32 Using Rust

### Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 ldr-dracula
```

This will open a screen asking you to select options.  In the latest esp-hal, ADC requires us to explicitly enable the unstable features. 

- So you select the option "Enable unstable HAL features."

In this exercise, we will be printing messages to the system console. In order to do that, we will enable the logging feature.

- So, scroll to "Flashing, logging and debugging (espflash)" and hit Enter.

- Then, Select "Use defmt to print messages".


Then save it by pressing "s" in the keyboard.

## Update cargo.toml

The nb crate simplifies non-blocking I/O (e.g., reading sensors, UART data) by returning nb::Result with a WouldBlock error when an operation isn't ready, allowing you to retry later without blocking. 

If you're wondering why we need it, the ADC's read_oneshot function returns an nb::Result and may return nb::Error::WouldBlock if it's not ready yet. Wrapping it with nb::block makes your code keep retrying until the ADC has finished its job and returns a proper result.

```toml
nb = "1.1.0"
```

### Setup the LED

We have done this before; just set GPIO 33, which is connected to the LED, as an output pin and initialize it to a Low state.  

```rust
let mut led = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());
```

### Configure ADC 

We will configure GPIO 4 as an ADC input pin, which is one of the ADC2 channels. We will apply an attenuation level of 11dB, allowing the ADC to measure input voltages ranging from 150 mV to ~2450 mV.

```rust
let adc_pin = peripherals.GPIO4;
let mut adc2_config = AdcConfig::new();
let mut pin = adc2_config.enable_pin(adc_pin, Attenuation::_11dB);
let mut adc2 = Adc::new(peripherals.ADC2, adc2_config);
```

### Oneshot read

The `read_oneshot` function starts a single ADC conversion on the specified pin. It is non-blocking and returns a 16-bit value wrapped in a Result. However, we will use `nb::block` to block until the conversion is complete, then we will proceed futher.
 
```rust
let pin_value: u16 = nb::block!(adc2.read_oneshot(&mut pin)).unwrap();
```

### Toggling LED
Once we get the digital value, turning the LED on or off is simple logic. We will turn on the LED by setting it to High if the pin value is greater than 3500 (you can adjust this threshold as needed). Otherwise, we will turn off the LED. This threshold of 3500 is typically reached when the room is dark, and the LDR's resistance(R2) is high, resulting in a higher voltage. 

```rust
if pin_value > 3500 {
    led.set_high();
} else {
    led.set_low();
}
```
If you're wondering how the LDR's high resistance leads to a higher voltage, I recommend re-reading the ["How It Works"](../how-it-works.md) section and experimenting with the simulator.

### Final code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use esp_hal::clock::CpuClock;
use esp_hal::delay::Delay;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::main;
use esp_println as _;

// ADC
use esp_hal::analog::adc::{Adc, AdcConfig, Attenuation};

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

    let mut led = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());

    let adc_pin = peripherals.GPIO4;
    let mut adc2_config = AdcConfig::new();
    let mut pin = adc2_config.enable_pin(adc_pin, Attenuation::_11dB);
    let mut adc2 = Adc::new(peripherals.ADC2, adc2_config);
    let delay = Delay::new();

    loop {
        let pin_value: u16 = nb::block!(adc2.read_oneshot(&mut pin)).unwrap();
        esp_println::println!("{}", pin_value);

        if pin_value > 3500 {
            led.set_high();
        } else {
            led.set_low();
        }

        delay.delay_millis(500);
    }
}
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `ldr-dracula` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/ldr-dracula
```
