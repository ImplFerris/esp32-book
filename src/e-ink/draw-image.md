# Draw image or shape on e-Paper (e-ink) Display using ESP32

Let's level up the game even further by drawing an image and a shape on the e-Paper display. We can achieve this using the embedded-graphics crate, and optionally, the tinybmp crate.

We've already explored the tinybmp crate in the OLED module section. It allows us to load a BMP file and display it on the screen. Alternatively, you can use a raw byte array of the image to achieve the same result.

The crate requires the image to be in BMP format. If your image is in another format, you will need to convert it to BMP. For example, you can use the following command on Linux to convert a PNG image to a monochrome BMP:

```sh
convert ferris.png -monochrome ferris.bmp
```

I have created the Ferris BMP file, which you can use for this exercise. Download it from [here](./images/ferris.bmp).

<img style="display: block; margin: auto;" alt="ferris bmp file" src="./images/ferris.bmp"/>


## Generate project using esp-generate
To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 e-ink-image
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
tinybmp = "0.6.0"
```

We've already covered the details of the other crates, as well as the basic setup for the SPI and display module code.  So, we won't go over those details again. Instead, let's jump straight into displaying an image and a shape after clearing the screen.

## Black background
Instead of using a white background, we'll fill the background with black for this project. I chose black because the image we're working with has a dark background, while the main subject is composed of white color. This will give better look.

```rust
// Clear any existing image
epd.clear_frame(&mut spi_dev, &mut Delay).unwrap();
display.clear(Color::Black).unwrap();
epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
    .unwrap();
Timer::after(Duration::from_secs(5)).await;
```

## Display the image
Place the ferris.bmp file inside the src folder. The code is pretty straightforward: load the image as bytes and pass it to the from_slice function of the Bmp. Then, you can use it with the Image.

```rust
let bmp_data = include_bytes!("../ferris.bmp");
let bmp = Bmp::from_slice(bmp_data).unwrap();
let image = Image::new(&bmp, Point::new(25, 60));
image.draw(&mut display).unwrap();

epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
    .unwrap();
Timer::after(Duration::from_secs(5)).await;
```

## Draw a shape
Let's draw a circle on the display.
```rust
// Display a circle
Circle::new(Point::new(80, 10), 40)
    .into_styled(PrimitiveStyle::with_stroke(Color::White, 2))
    .draw(&mut display)
    .ok();

epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
    .unwrap();
Timer::after(Duration::from_secs(5)).await;
```

## Clone the existing project
You can clone (or refer) project I created and navigate to the `e-ink-image` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/e-ink-image/
```

## The full code

```rust
#![no_std]
#![no_main]

use defmt::info;
use embassy_executor::Spawner;
use embassy_time::{Delay, Duration, Timer};
use embedded_graphics::image::Image;
use embedded_graphics::prelude::*;
use embedded_graphics::primitives::{Circle, PrimitiveStyle};
use epd_waveshare::prelude::WaveshareDisplay;
use esp_hal::clock::CpuClock;
use esp_hal::timer::timg::TimerGroup;
use esp_println as _;

use embedded_hal_bus::spi::ExclusiveDevice;
use epd_waveshare::color::Color;
use epd_waveshare::epd1in54_v2::{Display1in54, Epd1in54};
use esp_hal::gpio::{Input, InputConfig, Level, Output, OutputConfig, Pull};
use esp_hal::spi::master::{Config as SpiConfig, Spi};
use esp_hal::spi::Mode as SpiMode;
use esp_hal::time::Rate;
use tinybmp::Bmp;

#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
    esp_println::println!("Panic occurred: {:?}", info);

    loop {}
}

#[esp_hal_embassy::main]
async fn main(_spawner: Spawner) {
    // generator version: 0.3.1

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timer0 = TimerGroup::new(peripherals.TIMG1);
    esp_hal_embassy::init(timer0.timer0);

    info!("Embassy initialized!");

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
    display.clear(Color::Black).unwrap();
    epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
        .unwrap();
    Timer::after(Duration::from_secs(5)).await;

    // Dispaly image
    let bmp_data = include_bytes!("../ferris.bmp");
    let bmp = Bmp::from_slice(bmp_data).unwrap();
    let image = Image::new(&bmp, Point::new(25, 60));
    image.draw(&mut display).unwrap();
    epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
        .unwrap();
    Timer::after(Duration::from_secs(5)).await;

    // Display a circle
    Circle::new(Point::new(80, 10), 40)
        .into_styled(PrimitiveStyle::with_stroke(Color::White, 2))
        .draw(&mut display)
        .ok();
    epd.update_and_display_frame(&mut spi_dev, display.buffer(), &mut Delay)
        .unwrap();
    Timer::after(Duration::from_secs(5)).await;

    epd.sleep(&mut spi_dev, &mut Delay).unwrap();

    loop {
        info!("Hello world!");
        Timer::after(Duration::from_secs(60)).await;
    }
}
```
