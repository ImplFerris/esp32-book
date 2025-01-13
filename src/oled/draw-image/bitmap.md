# Using Bitmap Image file

You can use BMP (.bmp) files directly instead of raw image data by utilizing the [tinybmp](https://docs.rs/tinybmp/latest/tinybmp/) crate. tinybmp is a lightweight BMP parser designed for embedded environments. While it is mainly intended for drawing BMP images to embedded_graphics DrawTargets, it can also be used to parse BMP files for other applications.  This is perfect for our purpose. 

## BMP file
The crate requires the image to be in BMP format. If your image is in another format, you will need to convert it to BMP. For example, you can use the following command on Linux to convert a PNG image to a monochrome BMP:

```sh
convert ferris.png -monochrome ferris.bmp
```

I have created the Ferris BMP file, which you can use for this exercise. Download it from [here](../images/ferris.bmp).

<img style="display: block; margin: auto;" alt="ferris bmp file" src="../images/ferris.bmp"/>

## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 oled-image
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
tinybmp = "0.6.0"

```

## Using the BMP File
Place the ferris.bmp file inside the src folder. The code is pretty straightforward: load the image as bytes and pass it to the from_slice function of the Bmp. Then, you can use it with the Image.

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

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embedded_graphics::{image::Image, prelude::Point};
use esp_backtrace as _;
use esp_hal::prelude::*;
use log::info;
use ssd1306::{
    mode::DisplayConfigAsync, prelude::DisplayRotation, size::DisplaySize128x64,
    I2CDisplayInterface, Ssd1306Async,
};

use embedded_graphics::prelude::*;
use tinybmp::Bmp;

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

    // Include the BMP file data.
    let bmp_data = include_bytes!("../ferris.bmp");

    // Parse the BMP file.
    let bmp = Bmp::from_slice(bmp_data).unwrap();

    let image = Image::new(&bmp, Point::new(32, 0));
    image.draw(&mut display).unwrap();
    display.flush().await.unwrap();

    loop {
        Timer::after(Duration::from_secs(1)).await;
    }
}

``` 
