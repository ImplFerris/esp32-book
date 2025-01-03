# Writing Rust code to smoothly rotate Servo motor with ESP32

## Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 servo-motor
```

## Update dependencies
Add the following crate to the Cargo.toml file
```toml
embedded-hal = "1.0.0"
```

Then import the SetDutyCycle trait in the main.rs file. This is required if we want to use the max_duty_cycle, set_duty_cycle functions
```rust
use embedded_hal::pwm::SetDutyCycle;
```

## Timer and PWM Channel

Let's initialize the timer with frequency of 50Hz and 12 bit resolution and configure the channel.

```rust
let mut servo = peripherals.GPIO33;

let ledc = Ledc::new(peripherals.LEDC);

let mut hstimer0 = ledc.timer::<HighSpeed>(timer::Number::Timer0);
hstimer0
    .configure(timer::config::Config {
        duty: timer::config::Duty::Duty12Bit,
        clock_source: timer::HSClockSource::APBClk,
        frequency: 50.Hz(),
    })
    .unwrap();

let mut channel0 = ledc.channel(channel::Number::Channel0, &mut servo);
channel0
.configure(channel::config::Config {
    timer: &hstimer0,
    duty_pct: 12,  // not important, we will change it in loop
    pin_config: channel::config::PinConfig::PushPull,
})
.unwrap();
```

## Helper function

We have already explained the code in the last chapter.

```rust
    let max_duty_cycle = channel0.max_duty_cycle() as u32;

    // Minimum duty (2.5%)
    // For 12bit -> 25 * 4096 /1000 => ~ 102
    let min_duty = (25 * max_duty_cycle) / 1000;
    // Maximum duty (12.5%)
    // For 12bit -> 125 * 4096 /1000 => 512
    let max_duty = (125 * max_duty_cycle) / 1000;
    // 512 - 102 => 410
    let duty_gap = max_duty - min_duty;
    
    fn duty_from_angle(deg: u32, min_duty: u32, duty_gap: u32) -> u16 {
        let duty = min_duty + ((deg * duty_gap) / 180);
        duty as u16
    }
```

## Rotation

In the main loop, we first rotate from 0 degrees to 180 degrees. We add a 10ms gap to reach its position, which is enough since we are moving in smaller steps. Then, we use the rev() function to reverse the range, so it goes from 180 back to 0.

```rust
    loop {
        for deg in 0..=180 {
            let duty = duty_from_angle(deg, min_duty, duty_gap);
            channel0.set_duty_cycle(duty).unwrap();
            delay.delay_millis(10);
        }
        delay.delay_millis(500);

        for deg in (0..=180).rev() {
            let duty = duty_from_angle(deg, min_duty, duty_gap);
            channel0.set_duty_cycle(duty).unwrap();
            delay.delay_millis(10);
        }
        delay.delay_millis(500);
    }
```

## The full code
```rust
#![no_std]
#![no_main]

use embedded_hal::pwm::SetDutyCycle;
use esp_backtrace as _;
use esp_hal::{
    delay::Delay,
    ledc::{
        channel::{self, ChannelIFace},
        timer::{self, TimerIFace},
        HighSpeed, Ledc,
    },
    prelude::*,
};

#[entry]
fn main() -> ! {
    let peripherals = esp_hal::init({
        let mut config = esp_hal::Config::default();
        config.cpu_clock = CpuClock::max();
        config
    });

    let mut servo = peripherals.GPIO33;

    let ledc = Ledc::new(peripherals.LEDC);

    let mut hstimer0 = ledc.timer::<HighSpeed>(timer::Number::Timer0);
    hstimer0
        .configure(timer::config::Config {
            duty: timer::config::Duty::Duty12Bit,
            clock_source: timer::HSClockSource::APBClk,
            frequency: 50.Hz(),
        })
        .unwrap();
    let mut channel0 = ledc.channel(channel::Number::Channel0, &mut servo);
    channel0
        .configure(channel::config::Config {
            timer: &hstimer0,
            duty_pct: 12,
            pin_config: channel::config::PinConfig::PushPull,
        })
        .unwrap();
    let delay = Delay::new();

    let max_duty_cycle = channel0.max_duty_cycle() as u32;

    // Minimum duty (2.5%)
    // For 12bit -> 25 * 4096 /1000 => ~ 102
    let min_duty = (25 * max_duty_cycle) / 1000;
    // Maximum duty (12.5%)
    // For 12bit -> 125 * 4096 /1000 => 512
    let max_duty = (125 * max_duty_cycle) / 1000;
    // 512 - 102 => 410
    let duty_gap = max_duty - min_duty;

    loop {
        for deg in 0..=180 {
            let duty = duty_from_angle(deg, min_duty, duty_gap);
            channel0.set_duty_cycle(duty).unwrap();
            delay.delay_millis(10);
        }
        delay.delay_millis(500);

        for deg in (0..=180).rev() {
            let duty = duty_from_angle(deg, min_duty, duty_gap);
            channel0.set_duty_cycle(duty).unwrap();
            delay.delay_millis(10);
        }
        delay.delay_millis(500);
    }
}

fn duty_from_angle(deg: u32, min_duty: u32, duty_gap: u32) -> u16 {
    let duty = min_duty + ((deg * duty_gap) / 180);
    duty as u16
}

```


## Clone the existing project
You can clone (or refer) project I created and navigate to the `servo-motor` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/servo-motor
```