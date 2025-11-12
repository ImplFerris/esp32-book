# Using Bitmap Image file

You can use BMP (.bmp) files directly instead of raw image data by utilizing the [tinybmp](https://docs.rs/tinybmp/latest/tinybmp/) crate. tinybmp is a lightweight BMP parser designed for embedded environments. While it is mainly intended for drawing BMP images to embedded_graphics DrawTargets, it can also be used to parse BMP files for other applications.  This is perfect for our purpose. 

## BMP file
The crate requires the image to be in BMP format. If your image is in another format, you will need to convert it to BMP. For example, you can use the following command on Linux to convert a PNG image to a monochrome BMP:

```sh
convert ferris.png -monochrome ferris.bmp
```

I have created the Ferris BMP file, which you can use for this exercise. Download it from [here](../images/ferris.bmp).

<img style="display: block; margin: auto;" alt="ferris bmp file" src="../images/ferris.bmp"/>

## Project base

We will copy the old-image project and work on top of that. 

```sh
git clone https://github.com/ImplFerris/esp32-projects
cp -r esp32-projects/old-image ~/YOUR_PROJECT_FOLDER/oled-bmp
```

## Update Cargo.toml

We need one more crate called "tinybmp" to load the bmp image.

```toml
tinybmp = "0.6.0"

```

## Using the BMP File

Place the "ferris.bmp" file inside the src folder. The code is pretty straightforward: load the image as bytes and pass it to the from_slice function of the Bmp. Then, you can use it with the Image.

```rust
// the usual boilerplate code goes here...

// Include the BMP file data.
let bmp_data = include_bytes!("../ferris.bmp");

// Parse the BMP file.
let bmp = Bmp::from_slice(bmp_data).unwrap();

// usual code:
let image = Image::new(&bmp, Point::new(32, 0));
image.draw(&mut display).unwrap();
display.flush().await.unwrap();
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `oled-bmp` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/oled-bmp
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
use embedded_graphics::{image::Image, prelude::Point, prelude::*};

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

    // Include the BMP file data.
    let bmp_data = include_bytes!("../ferris.bmp");

    // Parse the BMP file.
    let bmp = tinybmp::Bmp::from_slice(bmp_data).unwrap();

    // usual code:
    let image = Image::new(&bmp, Point::new(32, 0));
    image.draw(&mut display).unwrap();
    display.flush().await.unwrap();

    loop {
        Timer::after(Duration::from_secs(1)).await;
    }
}
``` 
