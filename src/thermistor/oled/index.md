# Displaying Temperature on an OLED Display

There's nothing new in this chapter. In most real scenarios, you won't be connected to a system and will want to display the temperature on something like an OLED display. So, instead of printing the temperature to the system console, we'll print it to the OLED display. 

I chose not to use the OLED display in the previous chapter so that you could still complete the thermistor exercise even if you didn't have one at the time. However, for this exercise, you will need an OLED display.

## Prerequisites

- Obviously, you should have completed the previous chapters on the thermistor.
- [OLED Display section](../../oled/index.md)

I won't go into detail about the code, as we've already explained most of it in the Prerequisites section.

## Hardware Requirments

- An OLED display: (0.96 Inch I2C/IIC 4-Pin, 128x64 resolution, SSD1306 chip)
- Jumper wires
- NTC 103 Thermistor: 10K OHM, 5mm epoxy coated disc
- 10kΩ Resistor: Used with the thermistor to form a voltage divider 

## Circuit to connect OLED, Thermistor with ESP32

<img style="display: block; margin: auto;" alt="esp32 oled thermistor" src="../images/connectint esp32 oled thermistor.png"/>

### Connecting the Thermistor to the ESP32
It's the same circuit we used in the previous chapter, so you can leave the setup as it is.

1. One side of the Thermistor is connected to `GND`.
2. The other side of the Thermistor is connected to `GPIO13` which is ADC2 pin of ESP32
3. A 10kΩ resistor is connected in series with the Thermistor to create a voltage divider between the Thermistor and `3.3v`. So connect one side of the resistor to the `3.3V` and other one to `GPIO13`

### Connecting the OLED to the ESP32

We'll connect GPIO pin 18 to SCL and GPIO pin 23 to SDA. The VCC pin of the OLED will be connected to the 3.3V pin of the ESP32, and the Ground pin will be connected to the ESP32's ground.

