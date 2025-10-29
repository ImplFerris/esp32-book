# Hello, Rat (Using the `mipidsi` crate)

In this chapter, we'll write a simple program to get started with Ratatui. We will only show the basics here to see how it works. This tutorial uses the newer `mipidsi` crate.

## Prerequisites

- You'll need a TFT Display for this chapter. If you haven't completed the [TFT Display chapter](../../tft-display/index.md), I recommend finishing that first and then coming back here. Since the circuit connections between the TFT Display and ESP32 are already explained there, we won't repeat those instructions here.​

- Ratatui has a nice [tutorial to get started](https://ratatui.rs/tutorials/hello-ratatui/) for building TUI apps. If you already know Ratatui, this exercise is pretty simple - just gluing things together. If you're new, check out the official Ratatui tutorials later to get familiar with the basics and build better UIs.


## Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 hello-rat
```

This will open a screen asking you to select options. Ratatui requires heap allocation, so we will enable "unstable" and "alloc" features. 

- First, select the option "Enable unstable HAL features."
- Select the option "Enable allocations via the esp-alloc crate."

Optionally, you can enable the logging feature also

- So, scroll to "Flashing, logging and debugging (espflash)" and hit Enter.
- Then, Select "Use defmt to print messages".

Just save it by pressing "s" in the keyboard.

## Dependencies

First, let's update the Cargo.toml file with the dependencies we need to control the TFT display. We've already gone over these earlier in the TFT display section. 

```toml
embedded-hal-bus = { version = "0.3" }
embedded-graphics = "0.8.1"
mipidsi = "0.9.0"
static_cell = "2.1.1"
```

> Note: while this example uses the `mipidsi` crate for interacting with the TFT Display, you can use `mousefood` with any display driver crate that implements the embedded-graphics traits.

Now, let's add the ratatui and mousefood crates, which let us use ratatui in an embedded environment. 

```toml
mousefood = { git = "https://github.com/j-g00da/mousefood", rev = "cc9f8fe372f09342537bc31a1355f77f2693d70b", default-features = false, features = [
  "fonts",
] }
ratatui = { version = "0.30.0-alpha.5", default-features = false }
```

For no_std support, you'll need the latest mousefood code from GitHub and a compatible ratatui alpha version for now. 

## Imports

Let's bring in all the necessary crates and modules for setting up and controlling our TFT display. We'll import everything for graphics rendering with `embedded-graphics`, SPI communication through `esp-hal`, and the `mipidsi` crate which uses the `ili9341` display driver. Finally, we'll include the ratatui related imports.

```rust
// Embedded Graphics related
use embedded_graphics::pixelcolor::Rgb565;
use embedded_graphics::prelude::*;

// ESP32 SPI + Display Driver bridge
use esp_hal::delay::Delay;
use esp_hal::spi::master::Config as SpiConfig;
use esp_hal::spi::master::Spi;
use esp_hal::spi::Mode as SpiMode;
use esp_hal::time::Rate; // For specifying SPI frequency
use embedded_hal_bus::spi::ExclusiveDevice;
use mipidsi::{Builder, models::ILI9341Rgb565, interface::SpiInterface, options::{Orientation, Rotation}};
// For managing GPIO state
use esp_hal::gpio::{Level, Output, OutputConfig};
use static_cell::StaticCell;

// For ratatui
use mousefood::{EmbeddedBackend, EmbeddedBackendConfig};
use ratatui::layout::{Constraint, Flex, Layout};
use ratatui::widgets::{Block, Paragraph, Wrap};
use ratatui::{style::*, Frame, Terminal};

static SPI_BUFFER: StaticCell<[u8; 512]> = StaticCell::new();
```

## Initialize TFT Display Driver

Let's initialize the TFT display by first setting up the SPI interface, then creating and configuring the display instance.

```rust
// Initialize SPI
  let spi = Spi::new(
      peripherals.SPI2,
      SpiConfig::default()
          .with_frequency(Rate::from_mhz(60))
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

  let mut buffer = SPI_BUFFER.init([0; 512]);

  let spi_dev = ExclusiveDevice::new_no_delay(spi, cs).unwrap();
  let interface = SPIInterface::new(spi_dev, dc, buffer);

  let mut display = Builder::new(
    ILI9341Rgb565,
    interface
  )
    .reset_pin(reset)
    .init(&mut Delay::new())
    .unwrap();
  
  display.set_orientation(
    Orientation::default().rotate(Rotation::Deg270)
  ).unwrap();
  display.clear(Rgb565::BLACK).unwrap();
```

## Backends in Ratatui

Ratatui doesn't directly communicate with your screen. Instead, it uses an intermediary called a "backend" to handle all the low-level terminal operations. Think of the backend as a translator between Ratatui's high-level drawing commands and the actual terminal or display hardware.

Ratatui supports different backends, which makes it flexible enough to work in various environments. By default, it uses Crossterm as its backend for traditional terminal emulators. However, this isn't applicable for embedded systems. We need a backend that works in embedded environments. That's where mousefood crates comes in, it provide us Embedded Backend.

This backend allows Ratatui to render to LCD screens on microcontrollers, e-ink displays, small OLED screens, and any other display hardware that has support for the embedded-graphics library.

<img style="display: block; margin: auto;" src="../images/ratatui-embedded-backend.svg" alt="Ratatui Embedded Backend"/>

You can find more details on the Ratatui backends [here](https://ratatui.rs/concepts/backends/).

To make Ratatui use the mousefood Embedded Backend, you first initialize the EmbeddedBackend with a mutable reference to your display and default configuration, like this:

```rust
let backend = EmbeddedBackend::new(&mut display, EmbeddedBackendConfig::default());
```

Then, you create the Ratatui Terminal with this backend:

```rust
let mut terminal = Terminal::new(backend).unwrap();
```

## Draw Function

We will define the draw function which sets up the layout and visual elements of our UI frame. We begin by creating an outer green-bordered block titled "ESP32 Dashboard" that surrounds the entire interface. Within this block, we organize the layout first vertically and then horizontally to structure the content areas. 

We then create a paragraph widget displaying the text "Rat(a tui) inside ESP32,".  We wrap this paragraph in a yellow-bordered block titled " impl Rust " and render it at the centered position we defined with our layouts.

```rust
fn draw(frame: &mut Frame) {

    let outer_block = Block::bordered()
        .border_style(Style::new().green())
        .title(" ESP32 Dashboard ");
    frame.render_widget(outer_block, frame.area());

    let vertical_layout = Layout::vertical([Constraint::Length(3)])
        .flex(Flex::Center)
        .split(frame.area());

    let horizontal_layout = Layout::horizontal([Constraint::Length(25)])
        .flex(Flex::Center)
        .split(vertical_layout[0]);

    let text = "Rat(a tui) inside ESP32";
    let paragraph = Paragraph::new(text.dark_gray())
        .wrap(Wrap { trim: true })
        .centered();

    let bordered_block = Block::bordered()
        .border_style(Style::new().yellow())
        .title(" impl Rust ");

    frame.render_widget(paragraph.block(bordered_block), horizontal_layout[0]);
}
```

## Rendering

We finally call Ratatui's draw method, passing our draw function to render the UI to the display.

```rust
loop {
  terminal.draw(draw).unwrap();
}
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

use static_cell::StaticCell;
use embedded_hal_bus::spi::ExclusiveDevice;
use esp_backtrace as _;

// ESP Stuff
use esp_hal::{
    delay::Delay,
    spi::{
        master::{
            Config as SpiConfig,
            Spi
        },
        Mode as SpiMode,
    },
    time::Rate,
    gpio::{
        Level,
        Output,
        OutputConfig
    },
    clock::CpuClock,
    main
};

// Embedded graphics stuff
use embedded_graphics::pixelcolor::Rgb565;
use embedded_graphics::prelude::*;

// TFT Screen stuff
use mipidsi::{Builder, models::ILI9342CRgb565, interface::SpiInterface, options::{Orientation, Rotation}};

// Mousefood stuff
use mousefood::{EmbeddedBackend, EmbeddedBackendConfig};
use ratatui::{layout::{Constraint, Flex, Layout}, widgets::{Block, Paragraph, Wrap}};
use ratatui::{style::*, Frame, Terminal};

extern crate alloc;
esp_bootloader_esp_idf::esp_app_desc!();

static SPI_BUFFER: StaticCell<[u8; 512]> = StaticCell::new();

#[main]
fn main() -> ! {
    esp_println::logger::init_logger_from_env();

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    esp_alloc::heap_allocator!(#[unsafe(link_section = ".dram2_uninit")] size: 98767);

    let spi = Spi::new(
        peripherals.SPI2,
        SpiConfig::default()
            .with_frequency(Rate::from_mhz(60))
            .with_mode(SpiMode::_0)
    )
        .unwrap()
        .with_sck(peripherals.GPIO18)
        .with_mosi(peripherals.GPIO23);

    let cs = Output::new(peripherals.GPIO5, Level::Low, OutputConfig::default());
    let dc = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());
    let reset = Output::new(peripherals.GPIO4, Level::Low, OutputConfig::default());

    let buffer = SPI_BUFFER.init([0; 512]);

    let spi_dev = ExclusiveDevice::new_no_delay(spi, cs).unwrap();
    let interface = SpiInterface::new(spi_dev, dc, buffer);

    let mut display = Builder::new(
        ILI9342CRgb565,
        interface
    )
        .reset_pin(reset)
        .init(&mut Delay::new())
        .unwrap();

    // CRITICAL: Set orientation BEFORE clearing and creating backend
    display.set_orientation(
        Orientation::default().rotate(Rotation::Deg270)
    ).unwrap();
    
    // Clear with the new orientation
    display.clear(Rgb565::BLACK).unwrap();

    // Now create the backend with the properly oriented display
    let backend = EmbeddedBackend::new(&mut display, EmbeddedBackendConfig::default());
    let mut terminal = Terminal::new(backend).unwrap();

    loop {
        terminal.draw(draw).unwrap();
    }
}

fn draw(frame: &mut Frame) {
    let outer_block = Block::bordered()
        .title_style(Style::new().green())
        .title("ESP32 Dashboard");

    frame.render_widget(outer_block, frame.area());

    let vertical_layout = Layout::vertical([Constraint::Length(3)])
        .flex(Flex::Center)
        .split(frame.area());

    let horizontal_layout = Layout::horizontal([Constraint::Length(25)])
        .flex(Flex::Center)
        .split(vertical_layout[0]);

    let text = "Vaishnav Sabari Girish";
    let paragraph = Paragraph::new(text.dark_gray())
        .wrap(Wrap { trim: true })
        .centered();

    let bordered_block = Block::bordered()
        .border_style(Style::new().yellow())
        .title("RataTUI");

    frame.render_widget(paragraph.block(bordered_block), horizontal_layout[0]);

}
```


## Clone the existing project
You can clone (or refer) project I created and navigate to the `hello-rat-2` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd hello-rat-2/
```

## Flash the program

Run the following command from your project folder to build and flash the program to your ESP32:

```rust
cargo run --release
```

You should now see the Ratatui interface displayed on your Display.

<img style="display: block; margin: auto;" src="../images/ratatui-on-esp32-embedded-rust-2.jpg" alt="Ratatui on ESP32 (Type 2)"/>
