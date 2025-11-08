# Rust Code to Write a File to an SD Card Using the ESP32

In this exercise, we will create (or overwrite) a file on an SD card and write "Hello, Ferris!" into it.

## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 sdcard-write
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

## For time parsing
jiff = { version = "0.2.16", default-features = false, features = ["static"] }
```

## TimeSource with RTC

Before diving into the writing process, let's create a TimeSource for the sdmmc crate. In the previous exercise, we only read from a file and used a dummy time source. However, in this exercise, we want to create a file and ensure its time metadata is updated accordingly. To do this, we'll create a time source that's close enough to real-time for our needs.

We'll use the onboard RTC for this purpose. While this isn't a perfect solution; since the RTC requires an initial time to be set (which we'll pass through an environment variable). It will also reset whenever the ESP32 is restarted. Alternatively, you can get the current time from NTP servers using a Wi-Fi connection.

We will create a struct SdTimeSource that implements the TimeSource trait and uses the Rtc to get the current time. We have to specify how many years have passed since the year 1970 (the Unix epoch). We will get this by subtracting 1970 from the current year. We will also need to specify the month and date information in zero-indexed format, and the time as it is.

```rust
static TZ: jiff::tz::TimeZone = jiff::tz::get!("America/New_York");

struct SdTimeSource {
    timer: Rtc<'static>,
}

impl SdTimeSource {
    fn new(timer: Rtc<'static>) -> Self {
        Self { timer }
    }

    fn current_time(&self) -> u64 {
        self.timer.current_time_us()
    }
}

static TZ: jiff::tz::TimeZone = jiff::tz::get!("America/New_York");

impl TimeSource for SdTimeSource {
    fn get_timestamp(&self) -> Timestamp {
        let now_us = self.current_time();

        // Convert to jiff Time
        let now = jiff::Timestamp::from_microsecond(now_us as i64).unwrap();
        let now = now.to_zoned(TZ.clone());

        Timestamp {
            year_since_1970: (now.year() - 1970).unsigned_abs() as u8,
            zero_indexed_month: now.month().wrapping_sub(1) as u8,
            zero_indexed_day: now.day().wrapping_sub(1) as u8,
            hours: now.hour() as u8,
            minutes: now.minute() as u8,
            seconds: now.second() as u8,
        }
    }
}
```

Next, we'll update the SD card initialization code to use SdTimeSource as the time source. We'll pass the current time through an environment variable, parse it using jiff, and set it as the current time for the RTC.


```rust
// Timer for sdcard
let rtc = Rtc::new(peripherals.LPWR);
let current_time_us: u64 = env!("CURRENT_TIME_US")
    .parse()
    .expect("Invalid microseconds");
rtc.set_current_time_us(current_time_us);

let sd_timer = SdTimeSource::new(rtc);

println!("Init SD card controller and retrieve card size...");
let sd_size = sdcard.num_bytes().unwrap();
println!("card size is {} bytes\r\n", sd_size);

// Now let's look for volumes (also known as partitions) on our block device.
// To do this we need a Volume Manager. It will take ownership of the block device.
let volume_mgr = VolumeManager::new(sdcard, sd_timer);

// Try and access Volume 0 (i.e. the first partition).
// The volume object holds information about the filesystem on that volume.
let volume0 = volume_mgr.open_volume(VolumeIdx(0)).unwrap();

let root_dir = volume0.open_root_dir().unwrap();
```

## Write to file

Let's open the file in ReadWriteCreateOrTruncate mode. If the file doesn't exist, this will create it. If it does exist, it will truncate the file, clearing any existing content. 

```rust
let mut my_file = root_dir
.open_file_in_dir(
    "FERRIS.TXT",
    embedded_sdmmc::Mode::ReadWriteCreateOrTruncate,
)
.unwrap();
```

Once the file is open, we can write the message into it and then flush it to ensure the data is saved.

```rust
let line = "Hello, Ferris!";
if let Ok(()) = my_file.write(line.as_bytes()) {
    my_file.flush().unwrap();
    println!("Written Data");
} else {
    println!("Not wrote");
}
```

To verify, you can either connect the SD card to your computer or re-run the SD card reading program we used earlier.

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `sdcard-write` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/sdcard-write
```

## How to run?

We will pass the current date and time as an environment variable. To do this, we will add a shell command before cargo run that gets the current date and time and passes it to cargo run. This might not work in shells like Fish or on Windows.

```sh
CURRENT_DATETIME="$(date '+%Y-%m-%d %H:%M:%S')" cargo run --release
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
use esp_hal::clock::CpuClock;
use esp_hal::timer::timg::TimerGroup;
use esp_println::{self as _, println};

// SPI
use embedded_hal_bus::spi::ExclusiveDevice;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::spi::{self, master::Spi};
use esp_hal::time::Rate;

// SD card reader
use embedded_sdmmc::{SdCard, TimeSource, Timestamp, VolumeIdx, VolumeManager};

// For time
use esp_hal::rtc_cntl::Rtc;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

struct SdTimeSource {
    timer: Rtc<'static>,
}

impl SdTimeSource {
    fn new(timer: Rtc<'static>) -> Self {
        Self { timer }
    }

    fn current_time(&self) -> u64 {
        self.timer.current_time_us()
    }
}

static TZ: jiff::tz::TimeZone = jiff::tz::get!("America/New_York");

impl TimeSource for SdTimeSource {
    fn get_timestamp(&self) -> Timestamp {
        let now_us = self.current_time();

        // Convert to jiff Time
        let now = jiff::Timestamp::from_microsecond(now_us as i64).unwrap();
        let now = now.to_zoned(TZ.clone());

        Timestamp {
            year_since_1970: (now.year() - 1970).unsigned_abs() as u8,
            zero_indexed_month: now.month().wrapping_sub(1) as u8,
            zero_indexed_day: now.day().wrapping_sub(1) as u8,
            hours: now.hour() as u8,
            minutes: now.minute() as u8,
            seconds: now.second() as u8,
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

    // TODO: Spawn some tasks
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

    // Timer for sdcard
    let rtc = Rtc::new(peripherals.LPWR);
    let current_time_us: u64 = env!("CURRENT_TIME_US")
        .parse()
        .expect("Invalid microseconds");
    rtc.set_current_time_us(current_time_us);

    let sd_timer = SdTimeSource::new(rtc);

    let sdcard = SdCard::new(spi_dev, Delay);

    println!("Init SD card controller and retrieve card size...");
    let sd_size = sdcard.num_bytes().unwrap();
    println!("card size is {} bytes\r\n", sd_size);

    // Now let's look for volumes (also known as partitions) on our block device.
    // To do this we need a Volume Manager. It will take ownership of the block device.
    let volume_mgr = VolumeManager::new(sdcard, sd_timer);

    // Try and access Volume 0 (i.e. the first partition).
    // The volume object holds information about the filesystem on that volume.
    let volume0 = volume_mgr.open_volume(VolumeIdx(0)).unwrap();

    let root_dir = volume0.open_root_dir().unwrap();

    let my_file = root_dir
        .open_file_in_dir(
            "FERRIS.TXT",
            embedded_sdmmc::Mode::ReadWriteCreateOrTruncate,
        )
        .unwrap();

    let line = "Hello, Ferris!";
    if let Ok(()) = my_file.write(line.as_bytes()) {
        my_file.flush().unwrap();
        println!("Written Data");
    } else {
        println!("Not wrote");
    }

    loop {
        Timer::after(Duration::from_secs(30)).await;
    }
}
```
