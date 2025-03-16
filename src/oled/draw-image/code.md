## Code

By now, i hope you understand how the image is represented in the byte array. Now, let's move on to the coding part.

## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 oled-image
```

This will open a screen asking you to select options. 

- Select the option "Enables Wi-Fi via the esp-wifi crate. Requires alloc".  It automatically selects the espa-alloc crate option also
- Select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.


## Update Cargo.toml

```toml
ssd1306 = { git = "https://github.com/rust-embedded-community/ssd1306.git", rev = "f3a2f7aca421fbf3ddda45ecef0dfd1f0f12330e", features = [
    "async",
] }
embedded-graphics = "0.8.1"
```

## Boilerplate: Initialize I2C and Display instance
We have already explained this part in the previous [chapter](../hello-rust/index.md).

```rust
let i2c_bus = esp_hal::i2c::master::I2c::new(
    peripherals.I2C0,
    esp_hal::i2c::master::Config::default().with_frequency(Rate::from_khz(400)),
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

```

## Draw Image

We have created a byte array constant to represent the Ohm symbol.

```rust
// 8x5 pixels
#[rustfmt::skip]
const IMG_DATA: &[u8] = &[
    0b00111000,
    0b01000100,
    0b01000100,
    0b00101000,
    0b11101110,
];
```

We will create a raw image using the ImageRaw::new function. We need to specify the image width(i.e 8) and pixel color format with turbofish syntax `::<>`. The height of the image will be calculated automatically based on the data length and format. Since the display module we are using is only of two colors, we will use the BinaryColor enum.

Then we will draw the image at the starting position of the display, which is Point zero (x = 0, y = 0). Finally, we will flush the data to the display module.

```rust
let raw_image = ImageRaw::<BinaryColor>::new(IMG_DATA, 8);

let image = Image::new(&raw_image, Point::zero());

image.draw(&mut display).unwrap();
display.flush().await.unwrap();
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `oled-image` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/oled-image
```

## The full code
```rust
#![no_std]
#![no_main]

use defmt::info;
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embedded_graphics::image::{Image, ImageRaw};
use embedded_graphics::pixelcolor::BinaryColor;
use embedded_graphics::prelude::Point;
use embedded_graphics::prelude::*;
use esp_hal::timer::timg::TimerGroup;
use esp_hal::{clock::CpuClock, time::Rate};
use esp_println as _;
use ssd1306::mode::DisplayConfigAsync;
use ssd1306::{
    prelude::DisplayRotation, size::DisplaySize128x64, I2CDisplayInterface, Ssd1306Async,
};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[rustfmt::skip]
const IMG_DATA: &[u8] = &[
    0b00111000,
    0b01000100,
    0b01000100,
    0b00101000,
    0b11101110,
];

#[esp_hal_embassy::main]
async fn main(_spawner: Spawner) {
    // generator version: 0.3.1

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timer0 = TimerGroup::new(peripherals.TIMG1);
    esp_hal_embassy::init(timer0.timer0);

    info!("Embassy initialized!");

    let i2c_bus = esp_hal::i2c::master::I2c::new(
        peripherals.I2C0,
        esp_hal::i2c::master::Config::default().with_frequency(Rate::from_khz(400)),
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

    let raw_image = ImageRaw::<BinaryColor>::new(IMG_DATA, 8);

    let image = Image::new(&raw_image, Point::zero());

    image.draw(&mut display).unwrap();
    display.flush().await.unwrap();

    loop {
        Timer::after(Duration::from_secs(1)).await;
    }
}

```
