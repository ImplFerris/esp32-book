# "Hello, Rust!" in LCD Display

In this program, we will just print "Hello, Rust!" text in the LCD display. 

## HD44780 Drivers
During my research, I came across many Rust crates for controll the LCD Display, but these two stood out as working well. In this program, we will start by using the `hd44780-driver` crate.
- [hd44780-driver](https://crates.io/crates/hd44780-driver) 
- [liquid_crystal](https://crates.io/crates/liquid_crystal) 


## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 lcd-hello
```

This will open a screen asking you to select options. 

- First, select the option "Enable unstable HAL features.". Only after this, you will be able to select the embassy framework support
- Select the option "Adds embassy framework support."

Just save it by pressing "s" in the keyboard.

## Update Cargo.toml

We will use the hd44780-driver crate; and use the latest version directly from the GitHub repository, and specify which Git commit using the rev attribute. To enable async support, we can specify the "embedded-hal-async" feature.


```toml
# hd44780-driver = "0.4.0"
hd44780-driver = { git = "https://github.com/JohnDoneth/hd44780-driver", rev = "9009f2c24771ba0a20f8f7534471c9869188f76c", features = [
    "embedded-hal-async",
] }
```

## Imports

First add the necessary imports:

```rust
// I2C
use esp_hal::i2c::master::Config as I2cConfig; // for convenience, importing as alias
use esp_hal::i2c::master::I2c;
use esp_hal::time::Rate;

// HD44780 Driver
use hd44780_driver::HD44780;
use hd44780_driver::memory_map::MemoryMap1602;
use hd44780_driver::setup::DisplayOptionsI2C;
```

### Initialize I2C

We have configured the I2C interface to connect the ESP32 with the LCD module. We have set the I2C bus to run at 400 kHz. We have assigned GPIO18 to SCL (Serial Clock Line) and GPIO23 to SDA (Serial Data Line). We have also enabled async operation.

```rust
let i2c_bus = I2c::new(
        peripherals.I2C0,
        // I2cConfig is alias of esp_hal::i2c::master::I2c::Config
        I2cConfig::default().with_frequency(Rate::from_khz(400)),
    )
    .unwrap()
    .with_scl(peripherals.GPIO18)
    .with_sda(peripherals.GPIO23)
    .into_async();
```

### Initialize Display

We need to specify the I2C address of the LCD module. The address is usually provided in the module’s specifications. The most common one is 0x27. It's best to confirm it using the datasheet or by scanning for I2C devices.

```rust
let i2c_address = 0x27;

let Ok(mut lcd) = HD44780::new(
    DisplayOptionsI2C::new(MemoryMap1602::new()).with_i2c_bus(i2c_bus, i2c_address),
    &mut embassy_time::Delay,
) else {
    panic!("failed to initialize display");
};
```


### Prepare the display
Let's reset the display to its default state, move the cursor to the start, and clear any existing characters on the display.

```rust
// Unshift display and set cursor to 0
lcd.reset(&mut embassy_time::Delay).unwrap();

// Clear existing characters
lcd.clear(&mut embassy_time::Delay).unwrap();
```

### Write Text to the LCD
Let's write two texts on both lines of the LCD module.

```rust

// Display the following string
lcd.write_str("impl Rust", &mut embassy_time::Delay)
    .unwrap();

// Move the cursor to the second line
lcd.set_cursor_xy((0, 1), &mut embassy_time::Delay).unwrap();

// Display the following string on the second line
lcd.write_str("Hello, Ferris!", &mut embassy_time::Delay)
    .unwrap();
```

## Clone the existing project
You can clone (or refer) project I created and navigate to the `lcd-hello` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/lcd-hello/
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

// HD44780 Driver
use hd44780_driver::HD44780;
use hd44780_driver::memory_map::MemoryMap1602;
use hd44780_driver::setup::DisplayOptionsI2C;

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

    let i2c_address = 0x27;

    let Ok(mut lcd) = HD44780::new(
        DisplayOptionsI2C::new(MemoryMap1602::new()).with_i2c_bus(i2c_bus, i2c_address),
        &mut embassy_time::Delay,
    ) else {
        panic!("failed to initialize display");
    };

    // Unshift display and set cursor to 0
    lcd.reset(&mut embassy_time::Delay).unwrap();

    // Clear existing characters
    lcd.clear(&mut embassy_time::Delay).unwrap();

    // Display the following string
    lcd.write_str("impl Rust", &mut embassy_time::Delay)
        .unwrap();

    // Move the cursor to the second line
    lcd.set_cursor_xy((0, 1), &mut embassy_time::Delay).unwrap();

    // Display the following string on the second line
    lcd.write_str("Hello, Ferris!", &mut embassy_time::Delay)
        .unwrap();

    loop {
        info!("Hello world!");
        Timer::after(Duration::from_secs(1)).await;
    }
}
```
