# Write Rust code to read a file from an SD card using the ESP32.

Let's create a simple program that reads a file from the SD card and outputs its content to the system console. Make sure the SD card is formatted with FAT32 and contains a file to read (for example, "FERRIS.TXT" with the content "Hello, World!"). 
 
## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 sdcard-read
```

This will open a screen asking you to select options. 

- Select the option "Enable unstable HAL features"
- Then, select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.

## Additional Crates required
Update your Cargo.toml to add these additional crate along with the existing dependencies.

```toml
# sd card driver
embedded-sdmmc = "0.9.0"
# To convert Spi bus to SpiDevice
embedded-hal-bus = "0.3.0"
```


### embedded-sdmmc

We will use this crate to read and write files on an SD card for embedded devices. It does not use alloc or collections to keep the memory footprint low.


### embedded-hal-bus

To understand why we need the "embedded-hal-bus" crate, we first need to understand the Embedded HAL (Hardware Abstraction Layer). Embedded HAL provides several traits that offer a standard way to control common peripherals like GPIO, PWM, and communication interfaces (such as I2C, SPI, and UART) on microcontrollers. This allows drivers to be compatible across multiple microcontrollers (e.g., ESP32, Raspberry Pi Pico).

When we want to communicate with an SD card using SPI, embedded-hal provides the `SpiBus` and `SpiDevice` traits to support bus sharing. SpiBus represents the entire bus, while SpiDevice represents a device on that bus. Microcontroller-specific HALs (e.g., esp-hal) usually implement the SpiBus trait, and device drivers like sdmmc implement the SpiDevice trait.

So, we need to get the SpiDevice from the SpiBus to use it with the SD card. This is where the embedded-hal-bus crate helps. It provides different implementations of SpiDevice, like CriticalSectionDevice, ExclusiveDevice, and others. We'll use the ExclusiveDevice, as it's the simplest way to get an SpiDevice from an SpiBus, and it's suitable when no other devices are sharing the SPI bus.

## Dummy Timesource 

When you work with files on your computer, you might notice that files and directories have creation and modification times, which track when changes were made. SD cards work the same way; when you create or modify files, the file's timestamp gets updated.

The sdmmc driver needs a time source to get the current time for these updates. It provides a TimeSource trait, which you need to implement and pass to the VolumeManager during initialization.

Since we're only going to read files and won't be using this functionality, we'll create a DummyTimeSource that implements the TimeSource trait.

```rust
/// Code from https://github.com/rp-rs/rp-hal-boards/blob/main/boards/rp-pico/examples/pico_spi_sd_card.rs
/// A dummy timesource, which is mostly important for creating files.
#[derive(Default)]
pub struct DummyTimesource();

impl TimeSource for DummyTimesource {
    // In theory you could use the RTC of the rp2040 here, if you had
    // any external time synchronizing device.
    fn get_timestamp(&self) -> Timestamp {
        Timestamp {
            year_since_1970: 0,
            zero_indexed_month: 0,
            zero_indexed_day: 0,
            hours: 0,
            minutes: 0,
            seconds: 0,
        }
    }
}
```

## Setting Up the SPI for the SD Card Reader
To communicate with the SD card reader, we will initialize the SPI instance using the SPI2 peripheral. SD cards require the SPI clock to operate between 100 kHz and 400 kHz. In this setup, we will configure the SPI clock to 400 kHz and map the necessary pins to GPIOs for proper communication.

The SCK (Serial Clock) will be assigned to GPIO14, MOSI (Master Out, Slave In) to GPIO15, and MISO (Master In, Slave Out) to GPIO2. Additionally, we will configure the CS (Chip Select) pin on GPIO13 and set its initial state to High.

```rust
let spi_bus = Spi::new(
    peripherals.SPI2,
    spi::master::Config::default()
        .with_frequency(Rate::from_khz(400))
        .with_mode(spi::Mode::_0),
)
.unwrap()
.with_sck(peripherals.GPIO18)
.with_mosi(peripherals.GPIO23)
.with_miso(peripherals.GPIO19)
.into_async();

let sd_cs = Output::new(peripherals.GPIO5, Level::High, OutputConfig::default());
```

Once the SPI is configured, we will create an SpiDevice for the SD card reader. To achieve this, we will use the ExclusiveDevice provided by the embedded-hal-bus crate.  Finally, we initialize the SD card.

```rust
let spi_dev = ExclusiveDevice::new(spi_bus, sd_cs, Delay).unwrap();
let sdcard = SdCard::new(spi_dev, Delay);
```

##  Print the size of the SD Card

Next, we will use the volume manager to retrieve the size of the SD card in bytes and print it.

```rust
println!("Init SD card controller and retrieve card size...");
let sd_size = sdcard.num_bytes().unwrap();
println!("card size is {} bytes\r\n", sd_size);
```

## Volume manager

The next step is to initialize the volume manager to handle partitions and file systems on the SD card.

```rust
// Now let's look for volumes (also known as partitions) on our block device.
// To do this we need a Volume Manager. It will take ownership of the block device.
let volume_mgr = VolumeManager::new(sdcard, DummyTimesource::default());
```

## Open the directory

We use the volume manager to open the first primary partition (VolumeIdx(0)) on the SD card. After that, we access the root directory of this partition.

```rust
// Try and access Volume 0 (i.e. the first partition).
// The volume object holds information about the filesystem on that volume.
let volume0 = volume_mgr.open_volume(VolumeIdx(0)).unwrap();

let root_dir = volume0.open_root_dir().unwrap();
```


### Open file

Let's open the "FERRIS.TXT" file in read only mode and read it. Make sure you have added this file to your SD card with some content from your system earlier.

```rust
let mut my_file = root_dir
    .open_file_in_dir("FERRIS.TXT", embedded_sdmmc::Mode::ReadOnly)
    .unwrap();
```

### Print file content

We will read the file until it reaches the end. We will convert each byte to a character and print it.

```rust
while !my_file.is_eof() {
    let mut buffer = [0u8; 32];

    if let Ok(n) = my_file.read(&mut buffer) {
        for b in &buffer[..n] {
            print!("{}", *b as char);
        }
    }
}
```


## Clone the existing project
You can also clone (or refer) project I created and navigate to the `sdcard-read` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/sdcard-read
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
use embassy_time::{Delay, Duration, Timer};
use embedded_hal_bus::spi::ExclusiveDevice;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::spi;
use esp_hal::time::Rate;
use esp_hal::timer::timg::TimerGroup;
use esp_hal::{clock::CpuClock, spi::master::Spi};
use esp_println::{self as _, print, println};

// SD card reader
use embedded_sdmmc::{SdCard, TimeSource, Timestamp, VolumeIdx, VolumeManager};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

/// Code from https://github.com/rp-rs/rp-hal-boards/blob/main/boards/rp-pico/examples/pico_spi_sd_card.rs
/// A dummy timesource, which is mostly important for creating files.
#[derive(Default)]
pub struct DummyTimesource();

impl TimeSource for DummyTimesource {
    // In theory you could use the RTC of the rp2040 here, if you had
    // any external time synchronizing device.
    fn get_timestamp(&self) -> Timestamp {
        Timestamp {
            year_since_1970: 0,
            zero_indexed_month: 0,
            zero_indexed_day: 0,
            hours: 0,
            minutes: 0,
            seconds: 0,
        }
    }
}

#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0);

    info!("Embassy initialized!");

    let _ = spawner;

    let spi_bus = Spi::new(
        peripherals.SPI2,
        spi::master::Config::default()
            .with_frequency(Rate::from_khz(400))
            .with_mode(spi::Mode::_0),
    )
    .unwrap()
    .with_sck(peripherals.GPIO18)
    .with_mosi(peripherals.GPIO23)
    .with_miso(peripherals.GPIO19)
    .into_async();

    let sd_cs = Output::new(peripherals.GPIO5, Level::High, OutputConfig::default());
    let spi_dev = ExclusiveDevice::new(spi_bus, sd_cs, Delay).unwrap();
    let sdcard = SdCard::new(spi_dev, Delay);

    println!("Init SD card controller and retrieve card size...");
    let sd_size = sdcard.num_bytes().unwrap();
    println!("card size is {} bytes\r\n", sd_size);

    // Now let's look for volumes (also known as partitions) on our block device.
    // To do this we need a Volume Manager. It will take ownership of the block device.
    let volume_mgr = VolumeManager::new(sdcard, DummyTimesource::default());

    // Try and access Volume 0 (i.e. the first partition).
    // The volume object holds information about the filesystem on that volume.
    let volume0 = volume_mgr.open_volume(VolumeIdx(0)).unwrap();

    let root_dir = volume0.open_root_dir().unwrap();

    let my_file = root_dir
        .open_file_in_dir("FERRIS.TXT", embedded_sdmmc::Mode::ReadOnly)
        .unwrap();

    while !my_file.is_eof() {
        let mut buffer = [0u8; 32];

        if let Ok(n) = my_file.read(&mut buffer) {
            for b in &buffer[..n] {
                print!("{}", *b as char);
            }
        }
    }

    loop {
        Timer::after(Duration::from_secs(30)).await;
    }
}
```
