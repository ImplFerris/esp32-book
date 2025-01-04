# Controlling servo motor with ESP32's Motor Control Pulse Width Modulator (MCPWM) peripheral

In the previous exercise, we used the LEDC peripheral of the ESP32 to control the servo motor. In this exercise, we will use the MCPWM to achieve the same. An introduction to the MCPWM, its operation, and the corresponding functions in esp-hal is provided [here](../core-concepts/pwm/mcpwm.md). Please read that chapter before proceeding further.


### Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 servo-mcpwm
```

This will open a screen asking you to select options. For now, we dont need to select any options. Just save it by pressing "s" in the keyboard.

### Clock Config
Let's create an instance of the peripheral clock. We have chosen 1 MHz as the base clock frequency for the PWM. The function will internally calculate the prescaler and divide the input clock, which is 160 MHz.

```rust
let clock_cfg = PeripheralClockConfig::with_frequency(1.MHz()).unwrap();
```

For the servo, we need to achieve a final PWM frequency of 50 Hz. So, we need to keep the base clock frequency as low as possible. We can go down to 625 kHz with the maximum prescaler value of 255, but we keep it at 1 MHz to make the calculations easier.

### Configure MCPWM and Pin

We will use the MCPWM0 peripheral, will select timer0 and operator0. Next, we will configure it to use GPIO33, and set the PWM signal to stay high until it reaches the timestamp value we specify during the PWM cycle.

```rust
let mut mcpwm = McPwm::new(peripherals.MCPWM0, clock_cfg);
// connect operator0 to timer0
mcpwm.operator0.set_timer(&mcpwm.timer0);
// connect operator0 to pin
let mut pwm_pin = mcpwm
.operator0
.with_pin_a(peripherals.GPIO33, PwmPinConfig::UP_ACTIVE_HIGH);
```

### Configure the Timer

To achieve a 50 Hz PWM signal for the servo with a 1 MHz clock, the timer needs to count 20,000 ticks in total. Since the timer counts from 0 to 19,999, the period is set to 19_999, which gives a total of 20,000 ticks.

```rust
let timer_clock_cfg = clock_cfg
    .timer_clock_with_frequency(19_999, PwmWorkingMode::Increase, 50.Hz())
    .unwrap();

mcpwm.timer0.start(timer_clock_cfg);
```

### Rotation of Servo's horn

To rotate the servo horn, we adjust the PWM signal's timestamp. The timestamp values correspond to the desired angles:

- For 0 degrees, we set the timestamp to 500 (2.5% of 20,000).
- For 90 degrees, we set the timestamp to 1500 (7.5% of 20,000).
- For 180 degrees, we set the timestamp to 2500 (12.5% of 20,000).

After each adjustment, we give enough delay to allow the servo to reach the specified position.

```rust
loop {
        // 0 degree (2.5% of 20_000 => 500)
        pwm_pin.set_timestamp(500);
        delay.delay(1500.millis());

        // 90 degree (7.5% of 20_000 => 1500)
        pwm_pin.set_timestamp(1500);
        delay.delay(1500.millis());

        // 180 degree (12.5% of 20_000 => 2500)
        pwm_pin.set_timestamp(2500);
        delay.delay(1500.millis());
    }
```

## The full code
```rust
#![no_std]
#![no_main]

use esp_backtrace as _;
use esp_hal::delay::Delay;
use esp_hal::mcpwm::operator::PwmPinConfig;
use esp_hal::mcpwm::timer::PwmWorkingMode;
use esp_hal::mcpwm::{McPwm, PeripheralClockConfig};
use esp_hal::prelude::*;

#[entry]
fn main() -> ! {
    let peripherals = esp_hal::init({
        let mut config = esp_hal::Config::default();
        config.cpu_clock = CpuClock::max();
        config
    });

    esp_println::logger::init_logger_from_env();

    let delay = Delay::new();
    let clock_cfg = PeripheralClockConfig::with_frequency(1.MHz()).unwrap();
    let mut mcpwm = McPwm::new(peripherals.MCPWM0, clock_cfg);

    // connect operator0 to timer0
    mcpwm.operator0.set_timer(&mcpwm.timer0);
    // connect operator0 to pin
    let mut pwm_pin = mcpwm
        .operator0
        .with_pin_a(peripherals.GPIO33, PwmPinConfig::UP_ACTIVE_HIGH);

    // start timer with timestamp values in the range of 0..=19999 and a frequency
    // of 50 Hz
    let timer_clock_cfg = clock_cfg
        .timer_clock_with_frequency(19_999, PwmWorkingMode::Increase, 50.Hz())
        .unwrap();
    mcpwm.timer0.start(timer_clock_cfg);

    loop {
        // 0 degree (2.5% of 20_000 => 500)
        pwm_pin.set_timestamp(500);
        delay.delay(1500.millis());

        // 90 degree (7.5% of 20_000 => 1500)
        pwm_pin.set_timestamp(1500);
        delay.delay(1500.millis());

        // 180 degree (12.5% of 20_000 => 2500)
        pwm_pin.set_timestamp(2500);
        delay.delay(1500.millis());
    }
}
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `servo-mcpwm` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/servo-mcpwm
```
