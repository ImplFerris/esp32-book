# Display "Hello, Rust!" on OLED with ESP32

This exercise serves as a simple introduction to the OLED display, so we'll keep it straightforward by displaying "Hello, Rust!" on the OLED screen.  


## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 hello-oled
```

This will open a screen asking you to select options. 

- Select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.

## Update Cargo.toml

```toml
ssd1306 = { git = "https://github.com/rust-embedded-community/ssd1306.git", rev = "f3a2f7aca421fbf3ddda45ecef0dfd1f0f12330e", features = [
    "async",
] }
embedded-graphics = "0.8.1"
```

## Initialize I2C
We initialize the I2C interface for communication between the ESP32 and the OLED display. The I2C bus is configured with a frequency of 400 kHz and a timeout of 100 bus clock cycles. We assign GPIO18 to the SCL (Serial Clock Line) and GPIO23 to the SDA (Serial Data Line) for I2C communication, and enable async operation for the interface.

```rust
let i2c0 = esp_hal::i2c::master::I2c::new(
    peripherals.I2C0,
    esp_hal::i2c::master::Config {
        frequency: 400.kHz(),
        timeout: Some(100),
    },
)
.with_scl(peripherals.GPIO18)
.with_sda(peripherals.GPIO23)
.into_async();
```

## Initialize ssd1306 driver

First, we will use the helper struct "I2CDisplayInterface" to create a preconfigured I2C interface for the display. Next, we will use the "Ssd1306Async" struct (for non-async, use "Ssd1306") and pass the interface instance we created, the display size, which is "DisplaySize128x64", and the display rotation. Since we don't want any rotation, we will set it to "DisplayRotation::Rotate0".

The ssd1306 crate supports three display modes: 
- BasicMode, which offers basic control with lower-level methods
- BufferedGraphicsMode, which uses a framebuffer for advanced drawing and integrates with embedded-graphics
- and TerminalMode, a bufferless mode designed for drawing text and setting cursor positions like a terminal.

We will use the BufferedGraphicsMode for this exercise. 

Next, we call the init() function to initialize and clear the display in graphics mode.

```rust
let interface = I2CDisplayInterface::new(i2c0);
// initialize the display
let mut display = Ssd1306Async::new(interface, DisplaySize128x64, DisplayRotation::Rotate0)
    .into_buffered_graphics_mode();
display.init().await.unwrap();
```

### Text Style and Position
We will use monospaced fonts to display text. The MonoTextStyleBuilder will help us create the text style, and we will use a 6x10 pixel font size. You can find other monospaced fonts [here](https://docs.rs/embedded-graphics/latest/embedded_graphics/mono_font/ascii/index.html).

If you are using a multi-color OLED display, you can specify different font colors. However, since we are using a monochrome display, we will use "BinaryColor::On" to set the text color to white. This simply turns on those pixels needed to display the text.

```rust
let text_style = MonoTextStyleBuilder::new()
    .font(&FONT_6X10)
    .text_color(BinaryColor::On)
    .build();

Text::with_baseline("Hello, Rust!", Point::new(0, 16), text_style, Baseline::Top)
    .draw(&mut display)
    .unwrap();
```

The baseline is an imaginary line that determines where the text is aligned. We set the baseline, with the x position at 0 and the y position at 16. We also specify how the text should be aligned within this space. Baseline Enum controls how the text is positioned on the OLED. For example, using Baseline::Top aligns the top of the text with the starting point, while Baseline::Bottom aligns the bottom of the text with the starting point. It also has other options like Middle, Alphabetic.

I recommend adjusting the point values and the Baseline value to see how it affects the appearance. The visual changes will provide a better clarity.

Next, we can draw the text on any thing that implements the DrawTarget trait. The ssd1306 BufferedGraphicsMode implements this trait, so we can pass the display as a mutable reference to the draw function.

## Flush

Finally, we call the `flush` function, which writes the data to the display. Only after this will the updated content appear on the OLED screen.

```rust
display.flush().await.unwrap();
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `hello-oled` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/hello-oled
```

## Full code

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embedded_graphics::{
    mono_font::{ascii::FONT_6X10, MonoTextStyleBuilder},
    pixelcolor::BinaryColor,
    prelude::Point,
    text::{Baseline, Text},
};
use esp_backtrace as _;
use esp_hal::prelude::*;
use log::info;
use ssd1306::{
    mode::DisplayConfigAsync, prelude::DisplayRotation, size::DisplaySize128x64,
    I2CDisplayInterface, Ssd1306Async,
};

use embedded_graphics::prelude::*;

#[main]
async fn main(_spawner: Spawner) {
    let peripherals = esp_hal::init({
        let mut config = esp_hal::Config::default();
        config.cpu_clock = CpuClock::max();
        config
    });

    esp_println::logger::init_logger_from_env();

    let timer0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG1);
    esp_hal_embassy::init(timer0.timer0);

    info!("Embassy initialized!");

    let i2c0 = esp_hal::i2c::master::I2c::new(
        peripherals.I2C0,
        esp_hal::i2c::master::Config {
            frequency: 400.kHz(),
            timeout: Some(100),
        },
    )
    .with_scl(peripherals.GPIO18)
    .with_sda(peripherals.GPIO23)
    .into_async();

    let interface = I2CDisplayInterface::new(i2c0);
    // initialize the display
    let mut display = Ssd1306Async::new(interface, DisplaySize128x64, DisplayRotation::Rotate0)
        .into_buffered_graphics_mode();
    display.init().await.unwrap();

    let text_style = MonoTextStyleBuilder::new()
        .font(&FONT_6X10)
        .text_color(BinaryColor::On)
        .build();

    Text::with_baseline("Hello, Rust!", Point::new(0, 16), text_style, Baseline::Top)
        .draw(&mut display)
        .unwrap();

    display.flush().await.unwrap();

    loop {
        Timer::after(Duration::from_secs(1)).await;
    }
}
```
