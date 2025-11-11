# Write Text on e-Paper (e-ink) Display using ESP32

Let's create a simple program to draw text on the display module using the [epd-waveshare](https://docs.rs/epd-waveshare/latest/epd_waveshare/) crate. However, at the time of writing, this crate did not work as expected with the 1.54-inch V2 display. To resolve this, I had to apply some patches, so we will be using our forked version for now. I haven't submitted a pull request yet, as further improvements are needed (partial updates are not working as expected). Nevertheless, our forked version will be sufficient for our exercises. Once the issues are resolved, this chapter will be updated to use the latest fixed version.

## Generate project using esp-generate
To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 e-ink-hello
```

This will open a screen asking you to select options. 

- First, select the option "Enable unstable HAL features."
- Select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.

## Update cargo.toml

```toml
epd-waveshare = { features = [
  "graphics",
], git = "https://github.com/ImplFerris/epd-waveshare", branch = "1in54_v2_fix" }
embedded-hal-bus = { version = "0.1" }
embedded-graphics = "0.8.1"
```

By now, you should be familiar with embedded-hal-bus and the embedded-graphics crates.  The embedded-hal crate provides standardized interfaces (like SPI, I2C) for microcontroller peripherals, letting developers write reusable drivers that work across any compatible hardware. Basically, we will use this to convert the SpiBus provided by esp-hal into the SpiDevice required by the epd-waveshare driver.

We will use embedded-graphics crate to render text, shapes, and images on the display.

## SPI Setup
Let's initialize the SPI device for communication between the ESP32 and the e-paper display. This follows the usual setup: first, we initialize the SPI bus and then convert it into an SPI device using embedded-hal-bus.

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
let cs = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());
let mut spi_dev = ExclusiveDevice::new(spi, cs, Delay);
```

## Initialize the e-paper display

Now, let's initialize the e-paper display by passing the SPI device along with the corresponding GPIO pins for control. 

```rust
// Initialize Display
let busy_in = Input::new(
    peripherals.GPIO22,
    InputConfig::default().with_pull(Pull::None),
);
let dc = Output::new(peripherals.GPIO17, Level::Low, OutputConfig::default());
let reset = Output::new(peripherals.GPIO16, Level::Low, OutputConfig::default());
let mut display = Display1in54::default();
let mut epd = Epd1in54::new(&mut spi_dev, busy_in, dc, reset, &mut Delay, None).unwrap();
```

## Clear Display

First, we clear the e-paper display's internal buffer, then fill the display with white. After that, we update the screen so the changes show up. Finally, we add a short delay to let the display settle.

```rust
epd.clear_frame(&mut spi_dev, &mut Delay).unwrap();
display.clear(Color::White).unwrap();
epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
    .unwrap();
Timer::after(Duration::from_secs(5)).await;
```

## Write Text

Now, let's finally write the text "impl Rust for ESP32" on the screen at position (x=3, y=100). After that, we update and refresh the display to show the text. We also add a short delay to ensure the changes take effect.

```rust
draw_text(&mut display, "impl Rust for ESP32", 3, 100);
epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
    .unwrap();
Timer::after(Duration::from_secs(5)).await;
```

We use a helper function called draw_text, which uses the embedded_graphics crate's Text API to write the text with a specified font size and black color into the display buffer.

```rust
fn draw_text(display: &mut Display1in54, text: &str, x: i32, y: i32) {
    let text_style = MonoTextStyleBuilder::new()
        .font(&FONT_10X20)
        .text_color(Color::Black)
        .build();

    Text::with_baseline(text, Point::new(x, y), text_style, Baseline::Top)
        .draw(display)
        .unwrap();
}
```

## Sleep time

One of the important precautions to follow when using an e-ink display module is to either put it into sleep mode or completely power it off after use. Otherwise, you'll end up damaging the display.

```rust
epd.sleep(&mut spi_dev, &mut Delay).unwrap();
```

## Clone the existing project
You can clone (or refer) project I created and navigate to the `e-ink-hello` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/e-ink-hello/
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
use embassy_executor::Spawner;
use embassy_time::{Delay, Duration, Timer};
use embedded_hal_bus::spi::ExclusiveDevice;
use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Input, InputConfig, Level, Output, OutputConfig, Pull};
use esp_hal::timer::timg::TimerGroup;
use esp_println as _;

// SPI
use esp_hal::spi;
use esp_hal::spi::master::Spi;
use esp_hal::time::Rate;

// epd
use epd_waveshare::color::Color;
use epd_waveshare::epd1in54_v2::{Display1in54, Epd1in54};
use epd_waveshare::prelude::WaveshareDisplay;

// embedded graphics
use embedded_graphics::mono_font::MonoTextStyleBuilder;
use embedded_graphics::mono_font::ascii::FONT_10X20;
use embedded_graphics::prelude::*;
use embedded_graphics::text::{Baseline, Text};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0);

    info!("Embassy initialized!");

    // TODO: Spawn some tasks
    let _ = spawner;

    let spi_bus = Spi::new(
        peripherals.SPI2,
        spi::master::Config::default()
            .with_frequency(Rate::from_mhz(4))
            .with_mode(spi::Mode::_0),
    )
    .unwrap()
    //CLK
    .with_sck(peripherals.GPIO18)
    //DIN
    .with_mosi(peripherals.GPIO23);

    let cs = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());
    let mut spi_dev = ExclusiveDevice::new(spi_bus, cs, Delay);

    // Initialize Display
    let busy_in = Input::new(
        peripherals.GPIO22,
        InputConfig::default().with_pull(Pull::None),
    );
    let dc = Output::new(peripherals.GPIO17, Level::Low, OutputConfig::default());
    let reset = Output::new(peripherals.GPIO16, Level::Low, OutputConfig::default());
    let mut display = Display1in54::default();
    let mut epd = Epd1in54::new(&mut spi_dev, busy_in, dc, reset, &mut Delay, None).unwrap();

    // Clear any existing image
    epd.clear_frame(&mut spi_dev, &mut Delay).unwrap();
    display.clear(Color::White).unwrap();
    epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
        .unwrap();
    Timer::after(Duration::from_secs(5)).await;

    draw_text(&mut display, "impl Rust for ESP32", 3, 100);
    epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
        .unwrap();
    Timer::after(Duration::from_secs(5)).await;

    epd.sleep(&mut spi_dev, &mut Delay).unwrap();

    loop {
        info!("Hello world!");
        Timer::after(Duration::from_secs(1)).await;
    }
}

fn draw_text(display: &mut Display1in54, text: &str, x: i32, y: i32) {
    let text_style = MonoTextStyleBuilder::new()
        .font(&FONT_10X20)
        .text_color(Color::Black)
        .build();

    Text::with_baseline(text, Point::new(x, y), text_style, Baseline::Top)
        .draw(display)
        .unwrap();
}
```
