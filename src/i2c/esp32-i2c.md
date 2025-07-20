# ESP32 I2C Bus Interfaces

The ESP32 includes two I2C bus interfaces, each of which can be configured as either a controller or a target. According to the ESP32 datasheet, one interface is part of the main system, while the other belongs to the low-power subsystem.

Both interfaces support standard mode (100 Kbit/s), fast mode (400 Kbit/s), and can reach speeds up to 5 MHz, though the actual performance depends on the strength of the pull-up resistors on the SDA and SCL lines. The ESP32 supports 7-bit and 10-bit addressing, including dual address mode.

Thanks to the flexible GPIO matrix, the SDA and SCL signals can be assigned to almost any GPIO pins.

## Using I2C in esp-hal

To use I2C with the esp-hal crate, we configure the interface and assign the pins manually. Here's an example of setting up I2C with GPIO18 as SCL and GPIO23 as SDA:

```rust
 let i2c_bus = esp_hal::i2c::master::I2c::new(
    peripherals.I2C0,
    esp_hal::i2c::master::Config::default().with_frequency(Rate::from_khz(400)),
)
.unwrap()
.with_scl(peripherals.GPIO18)
.with_sda(peripherals.GPIO23);
```

Once the bus is ready, we can pass it to a driver, such as one for an OLED display:

```rust
let interface = I2CDisplayInterface::new(i2c_bus);
// initialize the display
let mut display = Ssd1306Async::new(interface, DisplaySize128x64, DisplayRotation::Rotate0)
    .into_buffered_graphics_mode();
```

