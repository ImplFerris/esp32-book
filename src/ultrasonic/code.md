## Writing Rust Code Use HC-SR04 Ultrasonic Sensor with ESP32

We'll start by generating the project using the template, then modify the code to fit the current project's requirements.


### Generate project using esp-generate

You have done this step already in the quick start section. 

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 ultrasonic
```

This will open a screen asking you to select options. In the latest esp-hal, ledc requires us to enable the unstable features. 

- So you select the option "Enable unstable HAL features."

Then save it by pressing "s" in the keyboard.

## Setup the LED Pin and configure PWM
You should understand this code by now. If not, please complete the Fading [LED section](../led/index.md) first.

Quick recap: Here, we're configuring the PWM for the LED, which allows us to control the brightness by adjusting the duty cycle.

```rust
// let led = peripherals.GPIO2; // uses onboard LED
    let led = peripherals.GPIO33;

// Configure LEDC
let mut ledc = Ledc::new(peripherals.LEDC);
ledc.set_global_slow_clock(LSGlobalClkSource::APBClk);
let mut lstimer0 = ledc.timer::<LowSpeed>(timer::Number::Timer0);
lstimer0
    .configure(timer::config::Config {
        duty: timer::config::Duty::Duty5Bit,
        clock_source: timer::LSClockSource::APBClk,
        frequency: Rate::from_khz(24),
    })
    .unwrap();
let mut channel0 = ledc.channel(channel::Number::Channel0, led);
channel0
    .configure(channel::config::Config {
        timer: &lstimer0,
        duty_pct: 10,
        pin_config: channel::config::PinConfig::PushPull,
    })
    .unwrap();
```

## Setup the Trigger Pin
We will configure GPIO 5 as an output pin with its initial state set to LOW. If you're wondering why it's an output, it's because we are sending a signal from the ESP32 to the ultrasonic module. This pin is connected to the Trig pin of the ultrasonic module.
 
```rust
let mut trig = Output::new(peripherals.GPIO5, Level::Low, OutputConfig::default());
```

## Setup the Echo Pin
We will configure GPIO 18 as an input pin since the ultrasonic module sends the signal back to the ESP32. The initial state of this pin will be set to Pull Down to ensure it starts in the low state.

```rust
let echo = Input::new(
    peripherals.GPIO18,
    InputConfig::default().with_pull(Pull::Down),
);
```

## 🦇 Light it Up 

### Step 1: Send the Trigger Pulse
We will set the trig pin to LOW so that we start fresh. We will set the trig pin to HIGH for 10 microseconds, then turn it back to LOW. This will trigger the module to send ultrasonic waves.

```rust
// Ensure the Trigger pin is low before starting
trig.set_low();
delay.delay_micros(2);

// Send a 10-microseconds high pulse
trig.set_high();
delay.delay_micros(10);
trig.set_low();
```

### Step 2: Measure the pulse width

Next, we will use two loops. The first loop will run as long as the echo pin state is LOW. Once it goes HIGH, we will record the current time in a variable. Then, we start the second loop, which will continue as long as the echo pin remains HIGH. When it returns to LOW, we will record the current time in another variable. The difference between these two times gives us the pulse width. 
 
```rust
 // Measure the duration the signal remains high
while echo.is_low() {}
let time1 = rtc.current_time();
while echo.is_high() {}
let time2 = rtc.current_time();
let pulse_width = match (time2 - time1).num_microseconds() {
    Some(pw) => pw as f64,
    None => continue,
};
```

### Step 3: Calculate the Distance

To calculate the distance, we need to use the pulse width. The pulse width tells us how long it took for the ultrasonic waves to travel to an obstacle and return. Since the pulse represents the round-trip time, we divide it by 2 to account for the journey to the obstacle and back.

The speed of sound in air is approximately 0.0343 cm per microsecond. By multiplying the time (in microseconds) by this value and dividing by 2, we obtain the distance to the obstacle in centimeters. 

```rust
let distance = (pulse_width * 0.0343) / 2.0;
```

### Step 4: PWM Duty cycle for LED
Finally, we adjust the LED brightness based on the measured distance.

The duty cycle percentage is calculated using our own logic, you can modify it to suit your needs. When the object is closer than 30 cm, the LED brightness will increase. The closer the object is to the ultrasonic module, the higher the calculated ratio will be, which in turn adjusts the duty cycle. This results in the LED brightness gradually increasing as the object approaches the sensor.
 
```rust
let duty_pct: u8 = if distance < 30.0 {
    let ratio = (30.0 - distance) / 30.0;
    let p = (ratio * 100.0) as u8;
    p.min(100)
} else {
    0
};

if let Err(e) = channel0.set_duty(duty_pct) {
    esp_println::println!("Failed to set duty cycle: {:?}", e);
}
```

### Full code

```rust
#![no_std]
#![no_main]

use esp_hal::clock::CpuClock;
use esp_hal::gpio::{InputConfig, OutputConfig};
use esp_hal::ledc::{LSGlobalClkSource, LowSpeed};
use esp_hal::main;
use esp_hal::time::Rate;

use esp_hal::{
    delay::Delay,
    gpio::{Input, Level, Output, Pull},
    ledc::{
        channel::{self, ChannelIFace},
        timer::{self, TimerIFace},
        Ledc,
    },
    rtc_cntl::Rtc,
};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[main]
fn main() -> ! {
    // generator version: 0.3.1

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    // let led = peripherals.GPIO2; // uses onboard LED
    let led = peripherals.GPIO33;

    // Configure LEDC
    let mut ledc = Ledc::new(peripherals.LEDC);
    ledc.set_global_slow_clock(LSGlobalClkSource::APBClk);
    let mut lstimer0 = ledc.timer::<LowSpeed>(timer::Number::Timer0);
    lstimer0
        .configure(timer::config::Config {
            duty: timer::config::Duty::Duty5Bit,
            clock_source: timer::LSClockSource::APBClk,
            frequency: Rate::from_khz(24),
        })
        .unwrap();
    let mut channel0 = ledc.channel(channel::Number::Channel0, led);
    channel0
        .configure(channel::config::Config {
            timer: &lstimer0,
            duty_pct: 10,
            pin_config: channel::config::PinConfig::PushPull,
        })
        .unwrap();

    // For HC-SR04 Ultrasonic
    let mut trig = Output::new(peripherals.GPIO5, Level::Low, OutputConfig::default());
    let echo = Input::new(
        peripherals.GPIO18,
        InputConfig::default().with_pull(Pull::Down),
    );

    let delay = Delay::new(); // We can use this since we are using unstable features

    let rtc = Rtc::new(peripherals.LPWR);

    loop {
        delay.delay_millis(5);

        // Trigger ultrasonic waves
        trig.set_low();
        delay.delay_micros(2);
        trig.set_high();
        delay.delay_micros(10);
        trig.set_low();

        // Measure the duration the signal remains high
        while echo.is_low() {}
        let time1 = rtc.current_time();
        while echo.is_high() {}
        let time2 = rtc.current_time();
        let pulse_width = match (time2 - time1).num_microseconds() {
            Some(pw) => pw as f64,
            None => continue,
        };

        // Derive distance from the pulse width
        let distance = (pulse_width * 0.0343) / 2.0;
        // esp_println::println!("Pulse Width: {}", pulse_width);
        // esp_println::println!("Distance: {}", distance);

        // Our own logic to calculate duty cycle percentage for the distance
        let duty_pct: u8 = if distance < 30.0 {
            let ratio = (30.0 - distance) / 30.0;
            let p = (ratio * 100.0) as u8;
            p.min(100)
        } else {
            0
        };

        if let Err(e) = channel0.set_duty(duty_pct) {
            // esp_println::println!("Failed to set duty cycle: {:?}", e);
            panic!("Failed to set duty cycle: {:?}", e);
        }

        delay.delay_millis(60);
    }
}
```

## Clone the existing project
You can clone (or refer) project I created and navigate to the `ultrasonic` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects/ultrasonic
cd esp32-projects/ultrasonic
``` 
