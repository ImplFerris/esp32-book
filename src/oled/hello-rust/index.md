# Display "Hello, Rust!" on OLED with ESP32

This exercise serves as a simple introduction to the OLED display, so we'll keep it straightforward by displaying "Hello, Rust!" on the OLED screen.  


## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 hello-oled
```

This will open a screen asking you to select options. 

- Select the option "Enable unstable HAL features"
- Then, select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.

Just save it by pressing "s" in the keyboard.


## Update Cargo.toml

```toml
ssd1306 = { version = "0.10.0", features = ["async"] }
embedded-graphics = "0.8.1"
```

## Imports

First add the necessary imports:

```rust
// I2C
use esp_hal::i2c::master::Config as I2cConfig; // for convenience, importing as alias
use esp_hal::i2c::master::I2c;
use esp_hal::time::Rate;

// OLED
use ssd1306::{I2CDisplayInterface, Ssd1306Async, prelude::*};

// Embedded Graphics
use embedded_graphics::{
    mono_font::{MonoTextStyleBuilder, ascii::FONT_6X10},
    pixelcolor::BinaryColor,
    prelude::Point,
    prelude::*,
    text::{Baseline, Text},
};
```

## Initialize I2C

We initialize the I2C interface for communication between the ESP32 and the OLED display. The I2C bus is configured with a frequency of 400 kHz and a timeout of 100 bus clock cycles. We assign GPIO18 to the SCL (Serial Clock Line) and GPIO23 to the SDA (Serial Data Line) for I2C communication, and enable async operation for the interface.

```rust
let i2c_bus = I2c::new(
        peripherals.I2C0,
        // I2cConfig is alias of esp_hal::i2c::master::I2c::Config
        I2cConfig::default().with_frequency(Rate::from_khz(400)),
    )
    .unwrap()
    .with_scl(peripherals.GPIO18)
    .with_sda(peripherals.GPIO23)
    .into_async();
```

## Initialize ssd1306 driver

Then, we will use the helper struct "I2CDisplayInterface" to create a preconfigured I2C interface for the display. Next, we will use the "Ssd1306Async" struct (for non-async, use "Ssd1306") and pass the interface instance we created, the display size, which is "DisplaySize128x64", and the display rotation. Since we don't want any rotation, we will set it to "DisplayRotation::Rotate0".

The ssd1306 crate supports three display modes: 
- BasicMode, which offers basic control with lower-level methods
- BufferedGraphicsMode, which uses a framebuffer for advanced drawing and integrates with embedded-graphics
- and TerminalMode, a bufferless mode designed for drawing text and setting cursor positions like a terminal.

We will use the BufferedGraphicsMode for this exercise. 

Next, we call the init() function to initialize and clear the display in graphics mode.

```rust
let interface = I2CDisplayInterface::new(i2c_bus);
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

The baseline is an imaginary line that determines where the text is aligned. We set the baseline, with the x position at 0 and the y position at 16. We also specify how the text should be aligned within this space. Baseline Enum controls how the text is positioned within the baseline. For example, using Baseline::Top aligns the top of the text with the starting point, while Baseline::Bottom aligns the bottom of the text with the starting point. It also has other options like Middle, Alphabetic.

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
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use defmt::info;
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};

use esp_hal::clock::CpuClock;
use esp_hal::timer::timg::TimerGroup;
use esp_println as _;

// I2C
use esp_hal::i2c::master::Config as I2cConfig; // for convenience, importing as alias
use esp_hal::i2c::master::I2c;
use esp_hal::time::Rate;

// OLED
use ssd1306::{I2CDisplayInterface, Ssd1306Async, prelude::*};

// Embedded Graphics
use embedded_graphics::{
    mono_font::{MonoTextStyleBuilder, ascii::FONT_6X10},
    pixelcolor::BinaryColor,
    prelude::Point,
    prelude::*,
    text::{Baseline, Text},
};

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

    let i2c_bus = I2c::new(
        peripherals.I2C0,
        // I2cConfig is alias of esp_hal::i2c::master::I2c::Config
        I2cConfig::default().with_frequency(Rate::from_khz(400)),
    )
    .unwrap()
    .with_scl(peripherals.GPIO18)
    .with_sda(peripherals.GPIO23)
    .into_async();

    let interface = I2CDisplayInterface::new(i2c_bus);
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
