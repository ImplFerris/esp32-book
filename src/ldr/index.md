## LDR (Light Dependent Resistor)

In this section, we will use an LDR (Light Dependent Resistor, also referred as photocell or photoresistor) with the ESP32. An LDR changes its resistance based on the amount of light falling on it. The brighter the light, the lower the resistance, and the dimmer the light, the higher the resistance. This makes it ideal for applications like light sensing, automatic lighting, or monitoring ambient light levels.

<img style="display: block; margin: auto;" alt="pico2" src="./images/ldr.png"/>


## Components Needed:
- LDR (Light Dependent Resistor)
- Resistor (typically 10kΩ); needed to create voltage divider
- Jumper wires (as usual)

## Prerequisite

To work with this, you should get familiar with what a voltage divider is and how it works. You also need to understand what ADC is and how it functions.
- [Voltage divider section](../core-concepts/voltage-divider.md)
- [ADC](../core-concepts/adc/index.md)
