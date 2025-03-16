# Access Point - Create Wi-Fi network on ESP32

So far, we have been using an existing Wi-Fi network. However, you can create your own Wi-Fi network with the ESP32 (just don't expect it to provide internet 😉). In this exercise, we will configure the ESP32 as an access point and run the web server.

## Generate project using esp-generate

We will create the project with Embassy support to take advantage of its async capabilities, making it a better fit for handling tasks that involve concurrency

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 wifi-ap
```

This will open a screen asking you to select options. 

- Select the option "Enables Wi-Fi via the esp-wifi crate. Requires alloc".  It automatically selects the espa-alloc crate option also
- Select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.


## Project structure

We will create two additional modules: web and wifi. These will be similar to what we implemented in the earlier sections while working on station mode. However, this time, the main difference is that we will configure the Wi-Fi to operate in Access Point mode. If you haven't completed the previous sections on Wi-Fi, it's highly recommended to finish them first.

```
├── build.rs
├── Cargo.toml
├── rust-toolchain.toml
├── src
│   ├── bin
│   │   └── main.rs
│   ├── lib.rs
│   ├── web.rs
│   └── wifi.rs
```


## Update dependencies

We will create a simple web server, just like in the previous exercises. Therefore, we will add the same dependency as before.

### picoserve crate
[picoserve](https://docs.rs/picoserve/latest/picoserve/) is a crate that provides an asynchronous HTTP server designed for bare-metal environments, heavily inspired by Axum. As you might have guessed from the name, it was first created with "Raspberry Pi Pico W" and embassy in mind. But it works fine with other embedded runtimes and hardware, including the ESP32. This crate makes our lives much easier. Without it, we would have to build the web server core from scratch, a time-consuming task that would be beyond the scope of this book.

```toml
picoserve = { version = "0.13.3", features = ["embassy"] }
```

### Task arena size update
We will update the embassy-executor with the task-arena-size-65536 feature. For more details, refer to the Task Arena Size documentation [here](https://docs.embassy.dev/embassy-executor/git/cortex-m/index.html#task-arena).

```toml
embassy-executor = { version = "0.6.3", features = ["task-arena-size-65536"] }
```

### Update the embassy-net
To make some functions compatible with the latest picoserve crate, I needed to update embassy-net to version 0.5.0.

```toml
embassy-net = { version = "0.5.0", features = [
    "tcp",
    "udp",
    "dhcpv4",
    "medium-ethernet",
] }
```

### Anyhow

This time, we will use anyhow::Error to handle errors in our code. You can achieve the same result without it, but I want to demonstrate how we can use anyhow for error handling. This library provides anyhow::Error, a trait object based error type that makes error handling in Rust applications easier and more idiomatic.

```rust
anyhow = { version = "1.0.95", default-features = false }
```

## Lib Module
We will define a macro to create a static variable that can be dynamically initialized and accessed across program functions. Additionally, we will declare the required modules in lib.rs. 

```rust
#![no_std]
#![feature(impl_trait_in_assoc_type)]

pub mod web;
pub mod wifi;

// When you are okay with using a nightly compiler it's better to use https://docs.rs/static_cell/2.1.0/static_cell/macro.make_static.html
#[macro_export]
macro_rules! mk_static {
    ($t:ty,$val:expr) => {{
        static STATIC_CELL: static_cell::StaticCell<$t> = static_cell::StaticCell::new();
        #[deny(unused_attributes)]
        let x = STATIC_CELL.uninit().write(($val));
        x
    }};
}
```


## Initialize Wi-Fi

First, I will import the library module and alias it as "lib". To access the modules defined within the library, we need to use the full project name. For consistency across different exercises, I will alias the module as "lib" in the import, so we can access them using "lib::web" instead of "wifi_ap::web" or "wifi_led::web" in different exercises.

```rust
use wifi_ap as lib;
```

To initialize the Wi-Fi controller, we first set up the necessary peripherals, including the timer, random number generator, and radio clock. 

```rust
let rng = Rng::new(peripherals.RNG);
let timg0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG0);
let wifi_init = lib::mk_static!(
    EspWifiController<'static>,
    esp_wifi::init(
        timg0.timer0,
        rng,
        peripherals.RADIO_CLK,
    )
    .unwrap()
);
```

## Spawn tasks
Next, we create the Wi-Fi stack by calling the start_wifi function, which we will define in the next chapter. This function starts the Wi-Fi connection and network tasks in the background. Additionally, we create a WebApp instance and spawn multiple web tasks based on the pool size. These web tasks are responsible for handling incoming web requests.


```rust
    // Configure and Start Wi-Fi tasks
    let stack = lib::wifi::start_wifi(wifi_init, peripherals.WIFI, rng, &spawner).await.unwrap();

    // Web Tasks
    let web = lib::web::WebApp::default();
    for id in 0..lib::web::WEB_TASK_POOL_SIZE {
        spawner.must_spawn(lib::web::web_task(id, *stack, web.app, web.config));
    }
    println!("Web server started...");
```
