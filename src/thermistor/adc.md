
## ADC to Resistance
When setting up the thermistor with the ESP32, we don't get the voltage directly. Instead, we receive an ADC value (refer to the [ADC](../core-concepts/adc/index.md) chapter). We need resistance value from the adc value for the thermistor temperature calculation(that will be discussed in the next chapters).

We will use this formula to calculate the resistance value from the ADC reading. If you need how it is derived, refer the [Deriving Resistance from ADC Value](./adc-maths.md).


<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">Methods</b>
        <p>There are other formulas and methods to calculate the temperature from the ADC value. This is the approach I felt aligns best with the voltage divider formula.</p>
    </div>
</div>

\\[
R_2 = \frac{R_1}{\left( \frac{\text{ADC_MAX}}{\text{adc_value}} - 1 \right)}
\\]

It can also be written like this:

\\[
R_2 = R_1 \left( \frac{\text{adc_value}}{\text{ADC_MAX} - \text{adc_value}} \right)
\\]

Where:
- **R2**: The resistance value of the thermistor that we need to calculate
- **R1**: The value of the resistor connected in series with the thermistor (typically 10kΩ)
- **ADC_MAX**: The maximum ADC value, 4095 (\\( 2^{12} - 1 \\)) for a 12-bit ADC
- **adc_value**: The value read from the ADC

### Rust Function

```rust

const ADC_MAX: f64 = 4095.0;
const R1_RES: f64 = 10_000.0; 

const fn adc_to_resistance(adc_value: f64) -> f64 {
    let x: f64 = adc_value/(ADC_MAX - adc_value);
    R1_RES * x
}

fn main() {
    let adc_value = 2000; // Our example ADC value;

    let r2 = adc_to_resistance(adc_value as f64);
    println!("Calculated Resistance (R2): {} Ω", r2);
}
```

<br/>

**Note:** 

If you connected the thermistor to power supply instead of GND. You will need opposite. since thermistor becomes R1.

\\[
R_1 = {R_2} \times \left(\frac{\text{ADC_MAX}}{\text{adc_value}} - 1\right)
\\]
