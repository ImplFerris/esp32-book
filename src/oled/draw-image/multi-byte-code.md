## Code

The main changes in the code are the image data and width. This will display the resistor symbol in the IEC-60617 style.


## Project base

We will copy the old-image project and work on top of that. 

```sh
git clone https://github.com/ImplFerris/esp32-projects
cp -r esp32-projects/old-image ~/YOUR_PROJECT_FOLDER/oled-rawimg
```


## Image Data

This time, we will use this to draw the resistor symbol on the OLED.

```rust
// 31x7 pixel
#[rustfmt::skip]
const IMG_DATA: &[u8] = &[
    // 1st row
    0b00000001,0b11111111,0b11111111,0b00000000,
    // 2nd row
    0b00000001,0b11111111,0b11111111,0b00000000,
    //3rd row
    0b00000001,0b10000000,0b00000011,0b00000000,
    //4th row
    0b11111111,0b10000000,0b00000011,0b11111110,
    //5th row
    0b00000001,0b10000000,0b00000011,0b00000000,
    //6th row
    0b00000001,0b11111111,0b11111111,0b00000000,
    //7th row
    0b00000001,0b11111111,0b11111111,0b00000000,
];
```

We need to set the width to 31. We'll draw the image at the point (x=35, y=35), though there's no particular reason for choosing these coordinates. I just wanted to show something other than the point zero. Feel free to experiment with different values for the point and explore other options.

```rust
 let raw_image = ImageRaw::<BinaryColor>::new(IMG_DATA, 31);

let image = Image::new(&raw_image, Point::new(35, 35));
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `oled-rawimg` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/oled-rawimg
```

## The full code

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
    image::{Image, ImageRaw},
    pixelcolor::BinaryColor,
    prelude::Point,
    prelude::*,
};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

// 31x7 pixel
#[rustfmt::skip]
const IMG_DATA: &[u8] = &[
    // 1st row
    0b00000001,0b11111111,0b11111111,0b00000000,
    // 2nd row
    0b00000001,0b11111111,0b11111111,0b00000000,
    //3rd row
    0b00000001,0b10000000,0b00000011,0b00000000,
    //4th row
    0b11111111,0b10000000,0b00000011,0b11111110,
    //5th row
    0b00000001,0b10000000,0b00000011,0b00000000,
    //6th row
    0b00000001,0b11111111,0b11111111,0b00000000,
    //7th row
    0b00000001,0b11111111,0b11111111,0b00000000,
];

#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0);

    info!("Embassy initialized!");

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

    let raw_image = ImageRaw::<BinaryColor>::new(IMG_DATA, 31);
    let image = Image::new(&raw_image, Point::new(35, 35));

    image.draw(&mut display).unwrap();
    display.flush().await.unwrap();

    loop {
        Timer::after(Duration::from_secs(1)).await;
    }
}
```
