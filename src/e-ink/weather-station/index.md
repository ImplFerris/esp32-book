# Write Rust code to build a Weather Dashboard using E-Paper and ESP32

Let's do something more fun with the e-ink display than just showing static text or images. We'll build a simple weather station that fetches real-time weather data from the internet using an HTTP API and updates the display with the latest info.  This is like a "Hello, World" program for the e-paper.


## Prerequisite

We'll be using the API from "openweathermap.org" to fetch the weather data. The website gives you a free API key with a limit of 1000 requests per day, which should be enough. Go to their website, sign up for a free account, and you'll get your own API key.


### Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 weather-station
```

This will open a screen asking you to select options. 

- First, select the option "Enable unstable HAL features."
- Select the option "Enable allocations via the esp-alloc crate."
- Now, you can enable "Enable Wi-Fi via esp-wifi crate."

Just save it by pressing "s" in the keyboard.


## Dependencies
Update your Cargo.toml to increase the task arena size for Embassy, and include the additional dependencies

```toml
embassy-executor = { version = "0.7.0", features = [
  "defmt",
  "task-arena-size-32768",
] }
```

**Additional dependencies:**

Let me give you a quick overview of the new dependencies. We've used `embedded-graphics` and `tinybmp` plenty of times already, including in the last chapter, so you should be pretty familiar with them by now.  We will use them to draw shapes and load bmp files. 

```toml
embedded-graphics = "0.8.1"
tinybmp = "0.6.0"
profont = "0.7.0"
chrono = { version = "0.4.40", default-features = false, features = ["serde"] }
serde = { version = "1.0.219", default-features = false, features = ["derive"] }
serde-json-core = "0.6.0"
serde_repr = "0.1.20"
reqwless = { default-features = false, features = [
  "esp-mbedtls",
  "log",
], git = "https://github.com/ImplFerris/reqwless", branch = "esp-hal-1.0.0" }
esp-mbedtls = { git = "https://github.com/esp-rs/esp-mbedtls.git", rev = "03458c3", features = [
  "esp32",
] }
epd-waveshare = { features = [
  "graphics",
], git = "https://github.com/ImplFerris/epd-waveshare"}
```


- [profont](https://docs.rs/profont/latest/profont/): The crate provides ProFont monospace programming font for embedded-graphics. We're using it because we need a slightly larger font size for the weather dashboard.
- [chrono](https://docs.rs/chrono/latest/chrono/): Used for handling date and time. We'll use it to deserialize the current datetime received from the API.
- [serde](https://docs.rs/serde/latest/serde/): A framework for serializing and deserializing Rust data structures. 
- [serde_json_core](https://docs.rs/serde-json-core/latest/serde_json_core/) : To deserialize JSON into a Rust struct, we'll use this crate. It's designed specifically for no_std environments—normally, you'd use serde_json instead.
- [serde_repr](https://docs.rs/serde_repr/latest/serde_repr/): The crate allows you to serialize and deserialize enums using their numeric values, making it easy to map numeric weather condition codes (like 200 or 201) from the API to Rust enum variants.
- [reqwless](https://docs.rs/reqwless/latest/reqwless/index.html): We've already used this crate, which is an HTTP client that works in a no_std environment. We'll use it to send requests and receive responses from the API. (Note: We're using a forked version of reqwless to make it work with esp-hal 1.0.0, since the original crate isn't compatible at the moment.)
- [esp-mbedtls](https://github.com/esp-rs/esp-mbedtls) : The reqwless crate supports two TLS backends: embedded-tls (the default) and esp-mbedtls. By default, reqwless uses embedded-tls, which supports only TLS 1.3. However, OpenWeatherMap doesn’t support TLS 1.3 at the moment. So, to make it work, we need to use the esp-mbedtls crate instead.
- [epd-waveshare](https://docs.rs/epd-waveshare/latest/epd_waveshare/) : A simple driver for Waveshare E-Ink displays over SPI. The original crate didn't work properly with the 1.54-inch variant, so I had to fork it and patch the code. It's more of a hack than a proper fix, so I haven't sent a PR to the original repo yet. For now, we will use the forked version.


## Project structure
This is the overall project structure, and we'll walk through it step by step. 

```
├── build.rs
├── src
│   ├── bin
│   │   └── main.rs
│   ├── ca_cert.pem
│   ├── dashboard.rs
│   ├── icons
│   │   ├── air.bmp
│   │   ├── ...bmp
│   │   └── ...bmp
│   ├── icons.rs
│   ├── lib.rs
│   ├── weather.rs
│   └── wifi.rs
├── ...

```

Since this will be a big chapter, I recommend referring to the finished project here (https://github.com/ImplFerris/esp32-epaper-weather/) for reference. You might need to check it for imports I won't cover them in the tutorial and other non-important details.

## The lib module 

In `lib.rs`, we define the submodules along with the helper macro we've been using in the Wi-Fi section. This macro reserves memory at compile time for a `static` value, but allows it to be initialized at runtime.

```rust
#![no_std]
pub mod dashboard;
pub mod icons;
pub mod weather;
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
