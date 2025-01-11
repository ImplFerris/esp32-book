# Writing Rust Code to Run a Website on ESP32

In earlier exercises, we accessed websites and printed the response in the console. In this exercise, we'll do the reverse: We will run a web server on the ESP32. This server will be accessible on the local Wi-Fi network. To make it accessible from the internet, additional setup is required. First, we'll focus on accessing the site within the local Wi-Fi network. We will still be working in STA mode only (i.e connecting to existing Wi-Fi).

## What We will Be Doing
We'll set up a simple web server to serve a single index.html page. For this example, let's assume the ESP32 has been assigned the IP address "192.168.0.101" (it gets displayed it in the console, when we are running). Once the server is running, you can access the page by navigating to 'http://192.168.0.101/' in your computer's browser. You can either use your own HTML page or the index.html page I created for this exercise, which you can find [here](https://github.com/ImplFerris/esp32-projects/blob/main/webserver-html/src/bin/index.html).

<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">DHCP vs Static IP </b>
        <p>You can set a static IP address instead of letting the DHCP server assign it. This makes the IP address consistent but adds some extra steps. To keep things simple, we won't do it in this exercise. We’ll show you how to set up a static IP in a later exercise.</p>
    </div>
</div>

No more waiting, let's start right away. 

## Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 webserver-html
```

This will open a screen asking you to select options. 

- Select the option "Enables Wi-Fi via the esp-wifi crate. Requires alloc".  It automatically selects the espa-alloc crate option also
- Select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.


## Update dependencies

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

## Project Structure

We will organize the logic by splitting it into modules. Under the lib, we will create two submodules: web.rs and wifi.rs.

```
├── build.rs
├── Cargo.toml
├── rust-toolchain.toml
├── src
│   ├── bin
│   │   └── async_main.rs
│   ├── index.html
│   ├── lib.rs
│   ├── web.rs
│   └── wifi.rs
```

## Lib Module

We'll relocate the mk_static macro, which allows creating static variables initialized at runtime, into the lib module. Additionally, we'll enable the impl_trait_in_assoc_type feature, as it is required by the picoserve crate.

```rust
#![no_std]
#![feature(impl_trait_in_assoc_type)]

pub mod web;
pub mod wifi;

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

## The main function (async_main.rs file)

To use the lib module, we would normally have to reference it using the full project name (e.g., webserver::web). However, to keep the references consistent across different exercises, we will alias the import as "lib". This allows us to use it as "lib::web" instead.

In the main function, we start with some boilerplate code to set up the global heap allocator and initialize Embassy.

Next, we create a Wi-Fi controller, which we will pass to the start_wifi function; that we will soon define in the wifi module. This function will return the network stack instance.

We will create a web application instance, configure routing and settings using the picoserve crate. We will then spawn multiple tasks to handle incoming requests based on the defined pool size. Each task receives the task ID, app instance, network stack, and server settings.

```rust
use webserver as lib;

#[main]
async fn main(spawner: Spawner) {
    let peripherals = esp_hal::init({
        let mut config = esp_hal::Config::default();
        config.cpu_clock = CpuClock::max();
        config
    });

    esp_alloc::heap_allocator!(72 * 1024);

    esp_println::logger::init_logger_from_env();

    let timer0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG1);
    esp_hal_embassy::init(timer0.timer0);

    let rng = Rng::new(peripherals.RNG);

    info!("Embassy initialized!");

    let timg0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG0);

    let wifi_init = lib::mk_static!(
        EspWifiController<'static>,
        esp_wifi::init(timg0.timer0, rng, peripherals.RADIO_CLK).unwrap()
    );

    let stack = lib::wifi::start_wifi(wifi_init, peripherals.WIFI, rng, &spawner).await;

    let web_app = lib::web::WebApp::default();
    for id in 0..lib::web::WEB_TASK_POOL_SIZE {
        spawner.must_spawn(lib::web::web_task(
            id,
            *stack,
            web_app.router,
            web_app.config,
        ));
    }
    println!("Web server started...");
}
```
