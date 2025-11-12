# How to Write Rust Code for Using Bluetooth Low Energy (BLE) on ESP32

Let's create a simple program to demonstrate Bluetooth Low Energy (BLE).  In this exercise, we will be using the `Trouble` crate with Embassy. We'll define a GATT service with two characteristics:

- One characteristic supports both read and write operations.  
- The other characteristic allows only read operations.  

I have generated UUIDs for these attributes (services and characteristics), and you can use the same ones.

> Refer to the Trouble repository for more examples: [https://github.com/embassy-rs/trouble/tree/main/examples](https://github.com/embassy-rs/trouble/tree/main/examples)

## Connecting to ESP32 Bluetooth

To interact with the ESP32's Bluetooth, we'll use the nRF Connect for Mobile app:  

🔗 [nRF Connect for Mobile](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile)  

This app lets us read and write data provided by the ESP32.


## Generate project using esp-generate

We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 bluetooth-low-energy
```

This will open a screen asking you to select options. 

- First, select the option "Enable unstable HAL features."
- Select the option "Enable allocations via the esp-alloc crate."
- Select the option "Adds embassy framework support".
- Now, you can enable "Enable BLE via the esp-radio crate (embassy-trouble)."

Just save it by pressing "s" in the keyboard.

## Update the Dependency

```toml
embassy-futures = "0.1.1"

# Requires for gatt_server macro to work
embassy-sync = { version = "0.7" }
```

## Initialize Wi-Fi controller

The ESP32 shares a single radio for both Wi-Fi and Bluetooth. In order to initialize Bluetooth, We will use the same Wi-Fi controller that we used for Wi-Fi.
 
```rust
let radio_init = esp_radio::init().expect("Failed to initialize Wi-Fi/BLE controller");
```

Let's initialize the Bluetooth connector. 

```rust
// find more examples https://github.com/embassy-rs/trouble/tree/main/examples/esp32
let transport = BleConnector::new(&radio_init, peripherals.BT, Default::default()).unwrap();
let ble_controller = ExternalController::<_, 20>::new(transport);

// let mut resources: HostResources<DefaultPacketPool, CONNECTIONS_MAX, L2CAP_CHANNELS_MAX> =
//     HostResources::new();
// let _stack = trouble_host::new(ble_controller, &mut resources);
```

After creating the BLE HCI controller, we will call the `run` function, which we will define shortly, to start the BLE stack.

```rust
ble::run(ble_controller).await;

loop {
    Timer::after(Duration::from_secs(5)).await;
}
```


## Full Code of main.rs

This is the complete code for the `main.rs` file. Next, we will create a module called `ble` a , where we will implement the run function along with helper functions for advertising and handling events.

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use bt_hci::controller::ExternalController;
use defmt::info;
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use esp_hal::clock::CpuClock;
use esp_hal::timer::timg::TimerGroup;
use esp_println as _;
use esp_radio::ble::controller::BleConnector;

// Our module
use bluetooth_low_energy as lib;
use lib::ble;

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[esp_rtos::main]
async fn main(_spawner: Spawner) -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    esp_alloc::heap_allocator!(#[unsafe(link_section = ".dram2_uninit")] size: 98767);

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0);

    info!("Embassy initialized!");

    let radio_init = esp_radio::init().expect("Failed to initialize Wi-Fi/BLE controller");
    // find more examples https://github.com/embassy-rs/trouble/tree/main/examples/esp32
    let transport = BleConnector::new(&radio_init, peripherals.BT, Default::default()).unwrap();
    let ble_controller = ExternalController::<_, 20>::new(transport);
    // let mut resources: HostResources<DefaultPacketPool, CONNECTIONS_MAX, L2CAP_CHANNELS_MAX> =
    //     HostResources::new();
    // let _stack = trouble_host::new(ble_controller, &mut resources);

    ble::run(ble_controller).await;

    loop {
        Timer::after(Duration::from_secs(5)).await;
    }
}
```
