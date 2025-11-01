# Buzzer Alert for Object Detection with ESP32 and Ultrasonic Sensor 

In our previous exercise, we used an LED that got brighter as the object got closer to the ultrasonic sensor module. Now, instead of the LED, we'll use an active buzzer. The buzzer will make a sound as the object moves closer. While you could use PWM to create different sounds or tones, we'll keep things simple for this project. As the object gets closer to the sensor, the buzzer will produce a beep sound.

## Circuit

The circuit is almost the same as before. The only difference is that you need to remove the LED and its associated resistor. Instead, connect the buzzer to GPIO 33. We will connect the positive pin(usually marked with plus sign) of the Buzzer to the GPIO 33 and other pin to the ground. 

<img style="display: block; margin: auto;" alt="hc-sr04 with buzzer and ESP32 circuit" src="./images/ESP32-HC-SR04-circuit-buzzer.png"/>


### Generate project using esp-generate

You have done this step already in the quick start section. 

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 ultrasonic-alert
```

This will open a screen asking you to select options. We don't need unstable or any other features. So just save it by pressing "s" in the keyboard.


## Code

We will set GPIO 33 as our output pin with an initial Low state. This is the same as the LED code; the only change is the variable name. 
   
```rust
let mut buzzer = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());
```

We won't need the timer or PWM configurations we used for the LED. Instead, we will set the buzzer to High (it will make a sound when it is High) if the distance is less than 30cm; otherwise, it will remain Low.

```rust
if distance < 30.0 {
    buzzer.set_high();
} else {
    buzzer.set_low();
}
```

## Full code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use esp_hal::clock::CpuClock;
use esp_hal::gpio::{Input, InputConfig, Level, Output, OutputConfig, Pull};
use esp_hal::main;
use esp_hal::time::{Duration, Instant};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[main]
fn main() -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let mut buzzer = Output::new(peripherals.GPIO33, Level::Low, OutputConfig::default());

    // For HC-SR04 Ultrasonic
    let mut trig = Output::new(peripherals.GPIO5, Level::Low, OutputConfig::default());
    let echo = Input::new(
        peripherals.GPIO18,
        InputConfig::default().with_pull(Pull::Down),
    );

    loop {
        blocking_delay(Duration::from_millis(5));

        // Trigger ultrasonic waves
        trig.set_low();
        blocking_delay(Duration::from_micros(2));
        trig.set_high();
        blocking_delay(Duration::from_micros(10));
        trig.set_low();

        // Measure the duration the signal remains high
        while echo.is_low() {}
        let time1 = Instant::now();
        while echo.is_high() {}
        let pulse_width = time1.elapsed().as_micros();

        // Derive distance from the pulse width
        let distance = (pulse_width as f64 * 0.0343) / 2.0;
        // esp_println::println!("Pulse Width: {}", pulse_width);
        // esp_println::println!("Distance: {}", distance);

        if distance < 30.0 {
            buzzer.set_high();
        } else {
            buzzer.set_low();
        }

        blocking_delay(Duration::from_millis(60));
    }
}

fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```


## Clone the existing project
You can clone (or refer) project I created and navigate to the `ultrasonic-alert` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/ultrasonic-alert
``` 
