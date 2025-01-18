# Rust Code to Write a File to an SD Card Using the ESP32

In this exercise, we will create (or overwrite) a file on an SD card and write "Hello, Ferris!" into it.


## TimeSource with RTC

Before diving into the writing process, let's create a TimeSource for the sdmmc crate. In the previous exercise, we only read from a file and used a dummy time source. However, in this exercise, we want to create a file and ensure its time metadata is updated accordingly. To do this, we'll create a time source that's close enough to real-time for our needs.

We'll use the onboard RTC for this purpose. While this isn't a perfect solution; since the RTC requires an initial time to be set (which we'll pass through an environment variable). It will also reset whenever the ESP32 is restarted. Alternatively, you can get the current time from NTP servers using a Wi-Fi connection.

We will create a struct SdTimeSource that implements the TimeSource trait and uses the Rtc to get the current time. We have to specify how many years have passed since the year 1970 (the Unix epoch). We will get this by subtracting 1970 from the current year. We will also need to specify the month and date information in zero-indexed format, and the time as it is.

```rust
struct SdTimeSource {
    timer: Rtc<'static>,
}

impl SdTimeSource {
    fn new(timer: Rtc<'static>) -> Self {
        Self { timer }
    }

    fn current_time(&self) -> chrono::NaiveDateTime {
        self.timer.current_time()
    }
}

impl TimeSource for SdTimeSource {
    fn get_timestamp(&self) -> Timestamp {
        let now = self.current_time();
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

Next, update the SD card initialization code to use SdTimeSource as the time source. We'll pass the current time through an environment variable, parse it using Chrono's NaiveDateTime, and set it as the current time for the RTC.


```rust
let rtc = Rtc::new(peripherals.LPWR);
const CURRENT_TIME: &str = env!("CURRENT_DATETIME");
let current_time = NaiveDateTime::parse_from_str(CURRENT_TIME, "%Y-%m-%d %H:%M:%S").unwrap();
rtc.set_current_time(current_time);

let sd_timer = SdTimeSource::new(rtc);

let sdcard = SdCard::new(spi, delay);
let mut volume_mgr = VolumeManager::new(sdcard, sd_timer);
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

use chrono::{Datelike, NaiveDateTime, Timelike};
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embedded_hal_bus::spi::ExclusiveDevice;
use embedded_sdmmc::{SdCard, TimeSource, Timestamp, VolumeIdx, VolumeManager};
use esp_backtrace as _;
use esp_hal::{
    delay::Delay,
    gpio::{Level, Output},
    prelude::*,
    rtc_cntl::Rtc,
    spi::{
        master::{Config, Spi},
        SpiMode,
    },
};
use esp_println::println;
use log::info;

struct SdTimeSource {
    timer: Rtc<'static>,
}

impl SdTimeSource {
    fn new(timer: Rtc<'static>) -> Self {
        Self { timer }
    }

    fn current_time(&self) -> chrono::NaiveDateTime {
        self.timer.current_time()
    }
}

impl TimeSource for SdTimeSource {
    fn get_timestamp(&self) -> Timestamp {
        let now = self.current_time();
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

    let delay = Delay::new();

    let spi = Spi::new_with_config(
        peripherals.SPI2,
        Config {
            frequency: 400.kHz(),
            mode: SpiMode::Mode0,
            ..Config::default()
        },
    )
    .with_sck(peripherals.GPIO18)
    .with_mosi(peripherals.GPIO23)
    .with_miso(peripherals.GPIO19);
    let sd_cs = Output::new(peripherals.GPIO5, Level::High);
    let spi = ExclusiveDevice::new(spi, sd_cs, delay).unwrap();

    let rtc = Rtc::new(peripherals.LPWR);
    const CURRENT_TIME: &str = env!("CURRENT_DATETIME");
    let current_time = NaiveDateTime::parse_from_str(CURRENT_TIME, "%Y-%m-%d %H:%M:%S").unwrap();
    rtc.set_current_time(current_time);

    let sd_timer = SdTimeSource::new(rtc);

    let sdcard = SdCard::new(spi, delay);
    let mut volume_mgr = VolumeManager::new(sdcard, sd_timer);

    println!("Init SD card controller and retrieve card size...");
    let sd_size = volume_mgr.device().num_bytes().unwrap();
    println!("card size is {} bytes\r\n", sd_size);

    let mut volume0 = volume_mgr.open_volume(VolumeIdx(0)).unwrap();
    let mut root_dir = volume0.open_root_dir().unwrap();

    let mut my_file = root_dir
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
