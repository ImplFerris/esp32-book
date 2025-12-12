# Deriving Resistance from ADC Value

> [!CAUTION]
> The mathematical derivation I presented in this section is currently under review. A user has raised a [GitHub issue](https://github.com/ImplFerris/esp32-book/issues/16) questioning the accuracy of this approach. It appears my initial research and derivation is incorrect. I have yet to research further and validate these formulas and will update this section once the analysis is complete. Please do not rely on this formula until then.

You can skip this chapter if you'd like. It simply explains the math behind deriving the resistance from the ADC value.  We combine the voltage divider formula with ADC Resolution formula to find the Resistance(R2). 

<span style="color:orange">Note:</span> It is assumed here that one side of the thermistor is connected to Ground (GND). I noticed that some online articles do the opposite, connecting one side of the thermistor to the power supply instead, which initially caused me some confusion.  When one side of the thermistor is connected to Ground and the other side to the ADC pin, it becomes R2 in the voltage divider.


<!-- 
**ADC voltage calculation formula**
\\[
V_{out} = {{V_{in}}} \times \frac{\text{adc_value}}{\text{ADC_MAX}}
\\] -->

**Voltage Divider Formula**
\\[
V_{out} = V_{in} \times \frac{R_2}{R_1 + R_2}
\\]


## Step 1: Combining the formulas

To calculate the voltage based on the ADC raw results, this formula can be used:

\\[
V_{out} = V_{in} \\times \\frac{\\text{adc_value}}{\\text{ADC_MAX}}
\\]

By substituting this relationship into the voltage divider formula:

\\[
V_{in} \\times \\frac{\\text{adc_value}}{\\text{ADC_MAX}} = V_{in} \\times \\frac{R_2}{R_1 + R_2}
\\]

---

## Step 2: Cancel out the Vin

We can cancel the Vin from both sides of the equation.

\\[
\require{cancel}
\cancel{V_{in}} \times \frac{\text{adc_value}}{\text{ADC_MAX}} = \cancel{V_{in}} \times \frac{R_2}{R_1 + R_2}
\\]

This simplifies to:

\\[
\\frac{\\text{adc_value}}{\\text{ADC_MAX}} = \\frac{R_2}{R_1 + R_2}
\\]

---

## Step 3: Getting \\( R_2 \\)

To solve for R2, we move the term (R1 + R2) to one side of the equation and rewrite it.

\\[
R_2 = (R_1 + R_2) \\times  \\frac{\\text{adc_value}}{\\text{ADC_MAX}} 
\\]

Expanding the right-hand side:

\\[
R_2 =  R_1 \\times \\frac{\\text{adc_value}}{\\text{ADC_MAX}} + R_2 \\times \\frac{\\text{adc_value}}{\\text{ADC_MAX}} 
\\]

Move R2 to one side of the equation to isolate it for further manipulation:
\\[
R_2 - (R_2 \\times \\frac{\\text{adc_value} }{\\text{ADC_MAX}} )= R_1  \\times \\frac{\\text{adc_value} }{\\text{ADC_MAX}}
\\]

Rearrange the equation to isolate R2:

\\[
R_2 \\left(1 - \\frac{\\text{adc_value}}{\\text{ADC_MAX}}\\right) = R_1 \\times \\frac{\\text{adc_value} }{\\text{ADC_MAX}} 
\\]

Expand the expression and cancel out the ADC_MAX in the denominator:

\\[
\require{cancel}
R_2 \\times \\frac{\\text{ADC_MAX} - \\text{adc_value}}{{\text{ADC_MAX}}} = R_1 \\times \\frac{\\text{adc_value} }{{\text{ADC_MAX}}} 
\\]
<br/>
\\[
\require{cancel}
R_2 \\times \\frac{\\text{ADC_MAX} - \\text{adc_value}}{\\cancel{\text{ADC_MAX}}} = R_1 \\times \\frac{\\text{adc_value} }{\\cancel{\text{ADC_MAX}}} 
\\]

So, we get:

\\[
R_2 \\times (\\text{ADC_MAX} - \\text{adc_value}) = R_1 \\times \\text{adc_value} 
\\]

Finally move (ADC_MAX−adc_value) to other side to get the R2:

\\[
R_2 = R_1 \times \\frac{\\text{adc_value}}{\\text{ADC_MAX} - \\text{adc_value}}
\\]

---

## Final Formula

The derived formula for R2 is:

\\[
R_2 = R_1 \times \\frac{\\text{adc_value}}{\\text{ADC_MAX} - \\text{adc_value}}
\\]

