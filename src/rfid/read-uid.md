# Read UID

Alright, let's get to the fun part and dive into some action! We'll start by writing a simple program to read the UID of the RFID tag.

## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 rfid-uid
```

This will open a screen asking you to select options. 

- Select the option "Enable unstable HAL features"
- Then, select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.

## Additional Crates required
Update your Cargo.toml to add these additional crate along with the existing dependencies.
```rust
mfrc522 = "0.8.0"
embedded-hal-bus = "0.2.0"
```

### mfrc522 Driver
We will be using the awesome crate  "[mfrc522](https://crates.io/crates/mfrc522)". It is still under development. However, it has everything what we need for purposes.


### embedded-hal-bus

To understand why we need the "embedded-hal-bus" crate, we first need to understand the Embedded HAL (Hardware Abstraction Layer). Embedded HAL provides several traits that offer a standard way to control common peripherals like GPIO, PWM, and communication interfaces (such as I2C, SPI, and UART) on microcontrollers. This allows drivers to be compatible across multiple microcontrollers (e.g., ESP32, Raspberry Pi Pico).

When we want to communicate with an RFID tag using SPI, embedded-hal provides the `SpiBus` and `SpiDevice` traits to support bus sharing. SpiBus represents the entire bus, while SpiDevice represents a device on that bus. Microcontroller-specific HALs (e.g., esp-hal) usually implement the SpiBus trait, and device drivers like mfrc522 implement the SpiDevice trait.

So, we need to get the SpiDevice from the SpiBus to use it with the SD card. This is where the embedded-hal-bus crate helps. It provides different implementations of SpiDevice, like CriticalSectionDevice, ExclusiveDevice, and others. We'll use the ExclusiveDevice, as it's the simplest way to get an SpiDevice from an SpiBus, and it's suitable when no other devices are sharing the SPI bus.
 

### Setting Up the SPI for the RFID Reader
To communicate with the RFID module, we will initialize the SPI instance using the SPI2 peripheral.  In this setup, we will configure the SPI clock to 5MHz and map the necessary pins to GPIOs for proper communication.

```rust
let spi_bus = Spi::new(
    peripherals.SPI2,
    spi::master::Config::default()
        .with_frequency(Rate::from_mhz(5))
        .with_mode(spi::Mode::_0),
)
.unwrap()
.with_sck(peripherals.GPIO18)
.with_mosi(peripherals.GPIO23)
.with_miso(peripherals.GPIO19)
.into_async();

let sd_cs = Output::new(peripherals.GPIO5, Level::High, OutputConfig::default());
```

### Getting the `SpiDevice` from SPI Bus
To work with the mfrc522 crate, we need an `SpiDevice`. Since we only have the SPI bus from ESP-HAL, we'll use the embedded_hal_bus crate to get the `SpiDevice` from the SPI bus.

```rust
let delay = Delay::new();
let spi_dev = ExclusiveDevice::new(spi_bus, sd_cs, delay).unwrap();
```

### Initialize the mfrc522
Next, we initialize the MFRC522 driver. To do this, we wrap the SpiDevice instance with the SpiInterface wrapper provided by the mfrc522 crate and pass it to the Mfrc522 initialization:

```rust
let spi_interface = SpiInterface::new(spi_dev);
let mut rfid = Mfrc522::new(spi_interface).init().unwrap();
```


### Helper Function to Print Byte Array as Hex String

We'll use this helper function to convert a u8 byte array (like a UID) into a printable hex string. This function will be used throughout the RFID exercises to display data. We might tweak it slightly depending on the specific requirements of each exercise.

```rust
fn print_hex_bytes(data: &[u8]) {
    for &b in data.iter() {
        print!("{:02x} ", b);
    }
    println!("");
}
```

### Read the UID and Print
The main logic for reading the UID is simple. We continuously send the REQA (Request A) command to check if a card or tag is nearby. If a card is present, it responds with the ATQA (Answer To reQuest code A).

The ATQA contains information about the tag's type, capabilities, and other details. Using the ATQA response, we select the tag and retrieve its UID.

```rust
loop {
    if let Ok(atqa) = rfid.reqa() {
        println!("Answer To reQuest code A");
        Timer::after(Duration::from_millis(50)).await;
        if let Ok(uid) = rfid.select(&atqa) {
            print_hex_bytes(uid.as_bytes());
            Timer::after(Duration::from_millis(500)).await;
        }
    }
}
```


## Clone the existing project
You can also clone (or refer) project I created and navigate to the `rfid-uid` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/rfid-uid
```


## Print the UID
After flashing the code onto the ESP32, bring the RFID tag close to the reader. The UID bytes will be displayed in the system console in hex format. Next, try the same with the key fob;it should display a different UID.

## Full code
```rust
#![no_std]
#![no_main]

use defmt::{info, println};
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embedded_hal_bus::spi::ExclusiveDevice;
use esp_hal::clock::CpuClock;
use esp_hal::delay::Delay;
use esp_hal::gpio::{Level, Output, OutputConfig};
use esp_hal::spi;
use esp_hal::spi::master::Spi;
use esp_hal::time::Rate;
use esp_hal::timer::timg::TimerGroup;
use esp_println::{self as _, print};
use mfrc522::comm::blocking::spi::SpiInterface;
use mfrc522::Mfrc522;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

fn print_hex_bytes(data: &[u8]) {
    for &b in data.iter() {
        print!("{:02x} ", b);
    }
    println!("");
}

#[esp_hal_embassy::main]
async fn main(_spawner: Spawner) {
    // generator version: 0.3.1

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let timer0 = TimerGroup::new(peripherals.TIMG1);
    esp_hal_embassy::init(timer0.timer0);

    info!("Embassy initialized!");

    let spi_bus = Spi::new(
        peripherals.SPI2,
        spi::master::Config::default()
            .with_frequency(Rate::from_mhz(5))
            .with_mode(spi::Mode::_0),
    )
    .unwrap()
    .with_sck(peripherals.GPIO18)
    .with_mosi(peripherals.GPIO23)
    .with_miso(peripherals.GPIO19);

    let sd_cs = Output::new(peripherals.GPIO5, Level::High, OutputConfig::default());

    let delay = Delay::new();
    let spi_dev = ExclusiveDevice::new(spi_bus, sd_cs, delay).unwrap();

    let spi_interface = SpiInterface::new(spi_dev);
    let mut rfid = Mfrc522::new(spi_interface).init().unwrap();

    loop {
        if let Ok(atqa) = rfid.reqa() {
            println!("Answer To reQuest code A");
            Timer::after(Duration::from_millis(50)).await;
            if let Ok(uid) = rfid.select(&atqa) {
                print_hex_bytes(uid.as_bytes());
                Timer::after(Duration::from_millis(500)).await;
            }
        }
    }
}
```
