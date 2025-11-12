# Write Text on TFT Display using ESP32

Let's create a simple program to draw text on the display module using the [ili9341](https://docs.rs/ili9341/0.6.0/ili9341/) crate. 

## Generate project using esp-generate
To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 tft-display-hello
```

This will open a screen asking you to select options. 

- First, select the option "Enable unstable HAL features."

Just save it by pressing "s" in the keyboard.

## Update cargo.toml

```toml
embedded-hal-bus = { version = "0.3" }
display-interface-spi = "0.5"
ili9341 = "0.6.0"
embedded-graphics = "0.8.1"
profont = "0.7.0"
```

By now, you should be familiar with embedded-hal-bus and the embedded-graphics crates.  The embedded-hal crate provides standardized interfaces (like SPI, I2C) for microcontroller peripherals, letting developers write reusable drivers that work across any compatible hardware. Basically, we will use this to convert the SpiBus provided by esp-hal into the SpiDevice.

However, unlike in previous chapters where we used SpiDevice directly, the ili9341 crate requires one more layer: it expects an interface that implements traits from the display-interface-spi crate. This crate defines traits and wrappers that bridge SPI bus drivers with display drivers by handling details like the data/command (DC) pin internally.

We use the profont crate to get a larger monospace font for our display, since the built-in embedded-graphics fonts are too small.

We will use embedded-graphics crate to render text, shapes, and images on the display.

## Required imports
```rust

// Usual imports
use defmt::info;
use esp_hal::clock::CpuClock;
use esp_hal::main;
use esp_hal::time::{Duration, Instant};
use esp_println as _;

// Embedded Grpahics related
use embedded_graphics::mono_font::MonoTextStyle;
use embedded_graphics::pixelcolor::Rgb565;
use embedded_graphics::prelude::*;
use embedded_graphics::text::{Baseline, Text};

// Larger font
use profont::{PROFONT_18_POINT, PROFONT_24_POINT};

// ESP32 SPI + Display Driver bridge
use display_interface_spi::SPIInterface;
use embedded_hal_bus::spi::ExclusiveDevice;
use esp_hal::delay::Delay;
use esp_hal::spi::master::Config as SpiConfig;
use esp_hal::spi::master::Spi;
use esp_hal::spi::Mode as SpiMode;
use esp_hal::time::Rate; // For specifying SPI frequency
use ili9341::{DisplaySize240x320, Ili9341, Orientation};

// For managing GPIO state
use esp_hal::gpio::{Level, Output, OutputConfig};
```

## SPI Setup
Let's initialize the SPI device for communication between the ESP32 and the display. This follows the usual setup: first, we initialize the SPI bus and then convert it into an SPI device using embedded-hal-bus.

```rust
// Initialize SPI
let spi = Spi::new(
    peripherals.SPI2,
    SpiConfig::default()
        .with_frequency(Rate::from_mhz(4))
        .with_mode(SpiMode::_0),
)
.unwrap()
//CLK
.with_sck(peripherals.GPIO18)
//DIN
.with_mosi(peripherals.GPIO23);
let cs = Output::new(peripherals.GPIO15, Level::Low, OutputConfig::default());
let dc = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());
let reset = Output::new(peripherals.GPIO4, Level::Low, OutputConfig::default());

let spi_dev = ExclusiveDevice::new_no_delay(spi, cs).unwrap();
let interface = SPIInterface::new(spi_dev, dc);
```

This time, we've added one more step: we create an SPIInterface using the display-interface-spi crate. This interface combines the SPI device and the data/command (DC) pin into a single abstraction. It simplifies communication by handling how commands and data are sent over SPI. We will pass this interface to the TFT display driver.

## Initialize the display

To initialize the display, we pass the SPI Interface, reset pin, delay, orientation, and screen size to the Ili9341 driver. This sets up everything the driver needs to start working with the display.

```rust
let mut display = Ili9341::new(
        interface,
        reset,
        &mut Delay::new(),
        Orientation::Portrait,
        DisplaySize240x320,
    )
    .unwrap();
```

We set the orientation to portrait, which means the display is treated as 240 pixels wide and 320 pixels tall. The display size is set to 240 by 320 pixels to match the screen's resolution. Together, these settings help the driver draw content correctly based on the shape and size of the display.

## Clear Display

Let's clear the display by filling its background with white. Since the TFT is a color display, we use the Rgb565 color format, which represents 16-bit color values (5 bits red, 6 bits green, 5 bits blue). This is the first time we are using Rgb565; until now, we have only worked with monochrome displays.

```rust
display.clear(Rgb565::WHITE).unwrap();
```

## Write Text

Now, let's finally display the text "impl Rust for ESP32" on the screen. We will write the two parts separately using different font sizes and colors.

```rust

let text_style = MonoTextStyle::new(&PROFONT_24_POINT, Rgb565::RED);
Text::with_baseline("impl Rust", Point::new(50, 150), text_style, Baseline::Top)
    .draw(&mut display)
    .unwrap();

let text_style = MonoTextStyle::new(&PROFONT_18_POINT, Rgb565::CSS_DIM_GRAY);

Text::with_baseline("for ESP32", Point::new(60, 180), text_style, Baseline::Top)
    .draw(&mut display)
    .unwrap();

```

We draw the first line, "impl Rust", using a red font, positioned 50 pixels from the left edge and 150 pixels down from the top of the screen. The second line, "for ESP32", is placed just below it, 60 pixels from the left and 180 pixels from the top. If you are writing different text, feel free to adjust the coordinates to achieve the alignment and spacing that looks best.
 
## Clone the existing project
You can clone (or refer) project I created and navigate to the `tft-display-hello` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/tft-display-hello/
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

use defmt::info;
use esp_hal::clock::CpuClock;
use esp_hal::main;
use esp_hal::time::{Duration, Instant};
use esp_println as _;

// Embedded Grpahics related
use embedded_graphics::mono_font::MonoTextStyle;
use embedded_graphics::pixelcolor::Rgb565;
use embedded_graphics::prelude::*;
use embedded_graphics::text::{Baseline, Text};

// Larger font
use profont::{PROFONT_18_POINT, PROFONT_24_POINT};

// ESP32 SPI + Display Driver bridge
use display_interface_spi::SPIInterface;
use embedded_hal_bus::spi::ExclusiveDevice;
use esp_hal::delay::Delay;
use esp_hal::spi::Mode as SpiMode;
use esp_hal::spi::master::Config as SpiConfig;
use esp_hal::spi::master::Spi;
use esp_hal::time::Rate; // For specifying SPI frequency
use ili9341::{DisplaySize240x320, Ili9341, Orientation};

// For managing GPIO state
use esp_hal::gpio::{Level, Output, OutputConfig};

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

    // Initialize SPI
    let spi = Spi::new(
        peripherals.SPI2,
        SpiConfig::default()
            .with_frequency(Rate::from_mhz(4))
            .with_mode(SpiMode::_0),
    )
    .unwrap()
    //CLK
    .with_sck(peripherals.GPIO18)
    //DIN
    .with_mosi(peripherals.GPIO23);
    let cs = Output::new(peripherals.GPIO15, Level::Low, OutputConfig::default());
    let dc = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());
    let reset = Output::new(peripherals.GPIO4, Level::Low, OutputConfig::default());

    let spi_dev = ExclusiveDevice::new_no_delay(spi, cs).unwrap();
    let interface = SPIInterface::new(spi_dev, dc);

    let mut display = Ili9341::new(
        interface,
        reset,
        &mut Delay::new(),
        Orientation::Portrait,
        DisplaySize240x320,
    )
    .unwrap();

    display.clear(Rgb565::WHITE).unwrap();

    let text_style = MonoTextStyle::new(&PROFONT_24_POINT, Rgb565::RED);
    Text::with_baseline("impl Rust", Point::new(50, 150), text_style, Baseline::Top)
        .draw(&mut display)
        .unwrap();

    let text_style = MonoTextStyle::new(&PROFONT_18_POINT, Rgb565::CSS_DIM_GRAY);

    Text::with_baseline("for ESP32", Point::new(60, 180), text_style, Baseline::Top)
        .draw(&mut display)
        .unwrap();

    loop {
        info!("Hello world!");
        let delay_start = Instant::now();
        while delay_start.elapsed() < Duration::from_millis(500) {}
    }
}
```
