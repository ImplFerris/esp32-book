## ESP32's LEDC peripheral to control servo motor

The ESP32 has LEDC and Motor Control Pulse Width Modulator (MCPWM) peripherals for PWM control. First, we will use the LEDC to generate the PWM, which we've already used in LED, buzzer, and other exercises.

In our previous exercises, we have been using the `set_duty` function, which takes the duty cycle percentage as a u8. But for the servo, we need fractional percentages (like 2.5%, 7.5%, 12.5%), which can't be represented with a u8.

### SetDutyCycle Trait
So, what do we do? Thankfully, the LEDC channels in esp-hal implement the embedded-hal trait SetDutyCycle, giving us more control over the PWM.

We will be using two functions in the SetDutyCycle trait: max_duty_cycle and set_duty_cycle.  
<br/>

**`max_duty_cycle`**:

This function will return the maximum duty cycle based on the duty resolution bits we configure in the timer. For example, if we set the resolution to 8 bits, the maximum duty cycle will be 256 (i.e., 2<sup>8</sup>). If we set it to 12 bits, the maximum duty cycle will be 4096 (i.e., 2<sup>12</sup>). We will be using a 12-bit resolution, so the maximum duty cycle will be 4096.

```rust
// We are converting to u32 (from u16) because we need u32 for the upcoming multiplication.
let max_duty_cycle = channel0.max_duty_cycle() as u32;
```
<br/>

**`set_duty_cycle`**:

This function is to set the duty cycle with in the range of 0 to 4096 for 12-bit resolution.

```rust
let duty = 512;
channel0.set_duty_cycle(duty).unwrap();
```

### But, How?
These functions take in or return u16 values. But how do we use percentage, which are in fractions? Instead of using the percentage directly, we calculate the corresponding value. For example, 2.5% of 4096 is approximately 102. This value is enough for us to move the servo motor to the correct position; in this case, it moves to 0 degrees.

For calculating the value from the percentage, we won't be using floats. Instead, we will cast the maximum duty cycle value to `u32`. 

Percentages like 2.5% can't be directly represented in `u32`, so we multiply the percentage by 10 to make it fit. For example, 2.5 becomes 25. Then, we divide the final value by 1000 (100 x 10) instead of 100.

\\[
\text{min\_duty} = \frac{Percent_{u32} \times \text{max\_duty\_cycle}}{1000}
\\]

For example:
```rust
// Minimum duty (2.5%) for servo position
// For 12bit -> 25 * 4096 /1000 => ~ 102
// it same as 2.5 *4096 / 100 => ~102
let min_duty = (25 * max_duty_cycle) / 1000;

// Maximum duty (12.5%) for servo position
// For 12bit -> 125 * 4096 /1000 => 512
let max_duty = (125 * max_duty_cycle) / 1000;

```

### Calculating Duty cycle from angle
We have a simple helper function that converts the angle to a duty cycle value. We have to pass the degree, min_duty, and the difference between the min_duty and max_duty of the servo position range. Then we cast the final value to u16 because the set_duty_cycle function accepts only u16.

```rust
let duty_gap = max_duty - min_duty; // 512 - 102 => 410
fn duty_from_angle(deg: u32, min_duty: u32, duty_gap: u32) -> u16 {
    let duty = min_duty + ((deg * duty_gap) / 180);
    duty as u16
}
```

For example, if the angle is 180 degree. We have already calculated min_duty and max_duty range which is 102 and 512, and the difference between them is 410. Let's substitute in the above equation.

duty = 102 + ((180 * 410) / 180) = 512

For the angle is 90 degree:

duty = 102 + ((90 * 410) / 180) = 307

307 is approximately 7.5% of 4096, which is what we needed for the 90 degree position.

### Rotation

Let's rotate the servo's horn from 0 degrees to 180, then back to 0 degrees in a loop. We first calculate the duty from the angle and set the duty cycle. We wait for 1500 milliseconds to allow the servo motor to reach its position. Try reducing the delay to 50ms, and you'll notice that the servo starts making jerky movements and doesn't reach the expected position at all.

```rust
loop{
    let duty = duty_from_angle(0, min_duty, duty_gap);
    channel0.set_duty_cycle(duty).unwrap();

    delay.delay_millis(1500); // allow to reach its position

    let duty = duty_from_angle(180, min_duty, duty_gap);
    channel0.set_duty_cycle(duty).unwrap();

    delay.delay_millis(1500); // allow to reach its position
}
```

Don't worry about how to run this code. In the next chapter, we'll look at a code that smoothly moves the servo from 0 to 180 degrees, then move back to 0 degrees in a loop.
