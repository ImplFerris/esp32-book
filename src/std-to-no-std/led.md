# LED

Next step is to configure the onboard LED of the DevKit v1. How are we going to do that? For that, we need to control a GPIO pin.

## GPIO 
GPIO stands for General Purpose Input/Output. These are pins on the microcontroller that can either read digital signals (input) or send digital signals (output).

- Want to read a button? Use a GPIO as input.

- Want to turn on an LED? Use a GPIO as output.

- Want to talk to a sensor or control a motor? GPIO pins are the starting point.

GPIOs are the most basic and most essential way your program talks to the outside world.

## What pin is the onboard LED connected to?

If you are using a bare ESP32 chip, there's no onboard LED.

But, we are using the ESP32 DevKit v1 development board. On this board, there is onboard LED which is connected to GPIO2. So that's the pin we'll use next.

## Initialize LED 

To control the onboard LED, we configure GPIO2 as an output pin. 

Adding this necessary import:

```rust
use esp_hal::gpio::{Level, Output, OutputConfig};
```

Add the following line inside the main function:

```rust
let mut led = Output::new(peripherals.GPIO2, Level::High, OutputConfig::default());
```

The peripherals.GPIO2 part gives us access to that GPIO2 pin. We set its initial state to Level::High, which turns the LED on. The OutputConfig::default() uses standard settings to make it an output pin.  Note here, we marked the led variable as mutable because we will change its state later.  We can use the toggle() function to switch the LED state between high and low, allowing us to blink the LED.

