## Print Joystick Movement ADC Values

In this program, we'll observe how joystick movement affects ADC values in real time. We will connect the ESP32 with the joystick. 

As you move the joystick, the corresponding ADC values will be printed in the system console. You can compare these values with the [previous Movement and ADC Diagram](./movement-and-12-bit-adc-value.md);they should approximately match the values shown. Pressing the joystick knob will print **"Button Pressed"** along with the current coordinates.


## Generate project using esp-generate
We will enable async (Embassy) support for this project.  To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 joystick-movement
```

This will open a screen asking you to select options. 

- Select the option "Enable unstable HAL features"
- Then, select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.

 
## Update cargo.toml

The nb crate simplifies non-blocking I/O (e.g., reading sensors, UART data) by returning nb::Result with a WouldBlock error when an operation isn't ready, allowing you to retry later without blocking. 

If you're wondering why we need it, the ADC's read_oneshot function returns an nb::Result and may return nb::Error::WouldBlock if it's not ready yet. Wrapping it with nb::block makes your code keep retrying until the ADC has finished its job and returns a proper result.

```toml
nb = "1.1.0"
```

### Configure ADC
Let's set up the ADC and configure GPIO 13 and GPIO 14, which are mapped to the VRX and VRY pins of the joystick: 

```rust
let mut adc2_config = AdcConfig::new();
let mut vrx_pin = adc2_config.enable_pin(peripherals.GPIO13, Attenuation::_11dB);
let mut vry_pin = adc2_config.enable_pin(peripherals.GPIO14, Attenuation::_11dB);

let mut adc2 = Adc::new(peripherals.ADC2, adc2_config);

```

We also configure GPIO15 as a pull-up input for the button: 

```rust
let btn = Input::new(
    peripherals.GPIO32,
    InputConfig::default().with_pull(Pull::Up),
);
```

### Printing Co-ordinates

We want to print the coordinates only when the vrx or vry values change beyond a certain threshold. This avoids continuously printing unnecessary values.

To achieve this, we initialize variables to store the previous values and a flag to determine when to print:

```rust
let mut prev_vrx: u16 = 0;
let mut prev_vry: u16 = 0;
let mut prev_btn_state = false;
let mut print_vals = true;
```

**Reading ADC Values:**

First, read the ADC values for vrx and vry. If there's an error during the read operation, we ignore it and continue the loop:

```rust
let Ok(vry): Result<u16, _> = nb::block!(adc2.read_oneshot(&mut vry_pin)) else {
    continue;
};
let Ok(vrx): Result<u16, _> = nb::block!(adc2.read_oneshot(&mut vrx_pin)) else {
    continue;
};
```

**Checking for Threshold Changes:**

Next, we check if the absolute difference between the current and previous values of vrx or vry exceeds a threshold (e.g., 100). If so, we update the previous values and set the print_vals flag to true:

```rust
if vrx.abs_diff(prev_vrx) > 100 {
    prev_vrx = vrx;
    print_vals = true;
}

if vry.abs_diff(prev_vry) > 100 {
    prev_vry = vry;
    print_vals = true;
}
```
Using a threshold filters out small ADC fluctuations, avoids unnecessary prints, and ensures updates only for significant changes.

**Printing the Coordinates**

If print_vals is true, we reset it to false and print the X and Y coordinates via the USB serial:

```rust
if print_vals {
    print_vals = false;
    println!("X: {} Y: {}\r\n", vrx, vry);
}
```

### Button Press Detection with State Transition
The button is normally in a high state. When you press the knob button, it switches from high to low. However, since the program runs in a loop, simply checking if the button is low could lead to multiple detections of the press. To avoid this, we only register the press once by detecting a high-to-low transition, which indicates that the button has been pressed.

To achieve this, we track the previous state of the button and compare it with the current state before printing the "button pressed" message. If the button is currently in a low state (pressed) and the previous state was high (not pressed), we recognize it as a new press and print the message. Then, we update the previous state to the current state, ensuring the correct detection of future transitions.

```rust
let btn_state = btn.is_low();
if btn_state && !prev_btn_state {
    println!("Button Pressed");
    print_vals = true;
}
prev_btn_state = btn_state;
```



## Clone the existing project
You can clone (or refer) project I created and navigate to the `joystick-movement` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/joystick-movement/
```

 
### The Full code

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
use esp_println::{self as _, println};

use esp_hal::analog::adc::{Adc, AdcConfig, Attenuation};
use esp_hal::gpio::{Input, InputConfig, Pull};

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

    let btn = Input::new(
        peripherals.GPIO32,
        InputConfig::default().with_pull(Pull::Up),
    );

    let mut adc2_config = AdcConfig::new();
    let mut vrx_pin = adc2_config.enable_pin(peripherals.GPIO13, Attenuation::_11dB);
    let mut vry_pin = adc2_config.enable_pin(peripherals.GPIO14, Attenuation::_11dB);

    let mut adc2 = Adc::new(peripherals.ADC2, adc2_config);

    let mut prev_vrx: u16 = 0;
    let mut prev_vry: u16 = 0;
    let mut prev_btn_state = false;
    let mut print_vals = true;

    loop {
        let Ok(vry): Result<u16, _> = nb::block!(adc2.read_oneshot(&mut vry_pin)) else {
            continue;
        };
        let Ok(vrx): Result<u16, _> = nb::block!(adc2.read_oneshot(&mut vrx_pin)) else {
            continue;
        };

        if vrx.abs_diff(prev_vrx) > 100 {
            prev_vrx = vrx;
            print_vals = true;
        }

        if vry.abs_diff(prev_vry) > 100 {
            prev_vry = vry;
            print_vals = true;
        }

        let btn_state = btn.is_low();
        if btn_state && !prev_btn_state {
            println!("Button Pressed");
            print_vals = true;
        }
        prev_btn_state = btn_state;

        if print_vals {
            print_vals = false;

            println!("X: {} Y: {}\r\n", vrx, vry);
        }

        Timer::after(Duration::from_millis(50)).await;
    }
}
```
