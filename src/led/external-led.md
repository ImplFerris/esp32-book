# Using external LED

You can do the same fading effect with external LED. 

## Hardware Requirements

- External LED
- Resistor (330 Ohms)
- Jumper wires (optional)
- Breadboard (optional) - You might need two breadboards to fit the ESP32 devkit properly, as it's quite wide. I bought two small breadboards and placed one side of the ESP32 on each.


## Circuit
- Connect the anode (longer leg) of the external LED to ESP32's GPIO 5 through the 330-ohm resistor
- Connect the cathode (shorter leg) of the LED  to the ground (GND) pin of the ESP32
<br/><br/>
<img style="display: block; margin: auto;" src="./images/esp32-external-led-circuit.jpg"/>


## Code changes

In the code, all you have to do is change the GPIO number from 2 to 5.

```rust
let led = peripherals.GPIO5;
```

## High speed channel
There's no fun in just changing one line. Let's use high-speed channel this time. To do that, we have to pass `HighSpeed` struct and update the clock source to use the `HSClockSource` enum.

```rust
let ledc = Ledc::new(peripherals.LEDC);

let mut hstimer0 = ledc.timer::<HighSpeed>(timer::Number::Timer0);
hstimer0
    .configure(timer::config::Config {
        duty: timer::config::Duty::Duty5Bit,
        clock_source: timer::HSClockSource::APBClk,
        frequency: 24.kHz(),
    })
    .unwrap();
```


## Clone the existing project
You can also clone (or refer) project I created and navigate to the `led-highfader` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/led-highfader
```
