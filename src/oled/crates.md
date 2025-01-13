# Crates 

We will primarily use two crates to control the OLED display: [ssd1306](https://docs.rs/ssd1306/latest/ssd1306/) and [embedded_graphics](https://docs.rs/embedded-graphics/latest/embedded_graphics/).


## SSD1306 OLED display driver

This crate offers a driver interface for the SSD1306 monochrome OLED display, supporting both I2C and SPI through the [display_interface](https://docs.rs/display-interface/latest/display_interface/) crate. It also has async support that you have to enable through the feature flag. 

### Add ssd1306 with Async support

At the time of writing, version 0.9.0 was causing issues. Although the problem has been fixed, the version hasn't been tagged yet. For now, we will use the GitHub commit.

```toml
# ssd1306 = { version = "0.9.0", features = ["async"] }
# https://github.com/rust-embedded-community/ssd1306/issues/219
ssd1306 = { git = "https://github.com/rust-embedded-community/ssd1306.git", rev = "f3a2f7aca421fbf3ddda45ecef0dfd1f0f12330e", features = [
    "async",
] }
```

### Add ssd1306 without Async support

```toml
# ssd1306 = { version = "0.9.0", features = ["async"] }
# https://github.com/rust-embedded-community/ssd1306/issues/219
ssd1306 = { git = "https://github.com/rust-embedded-community/ssd1306.git", rev = "f3a2f7aca421fbf3ddda45ecef0dfd1f0f12330e", features = [] }
```

## Embedded Graphics

To display text or draw images on the OLED display, we will use the embedded_graphics crate in combination with the ssd1306 crate.

"Embedded-graphics is a 2D graphics library that is focused on memory constrained embedded devices."

"A core goal of embedded-graphics is to draw graphics without using any buffers; the crate is no_std compatible and works without a dynamic memory allocator, and without pre-allocating large chunks of memory. To achieve this, it takes an Iterator based approach, where pixel colors and positions are calculated on the fly, with the minimum of saved state. This allows the consuming application to use far less RAM at little to no performance penalty."

You can use this crate with various OLED displays and drivers when working with different types of OLED modules. The [documentation](https://docs.rs/embedded-graphics/latest/embedded_graphics/) provides a detailed explanation of the features and supported drivers. I recommend going through it.


```toml
embedded-graphics = "0.8.1"
```
