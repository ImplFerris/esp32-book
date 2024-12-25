
# Pulse Width Modulation (PWM)

<style>
  
  .slider-container {
    margin: 20px 0;
  }

  label {
    margin-right: 10px;
  }

  .led-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2px;
    position: relative;
  }

  .led-body {
    width: 30px;
    height: 40px;
    background: radial-gradient(circle at center, #ff5555, #cc0000);
    border-radius: 50% 50% 0 0;
    border: 2px solid #990000;
    position: relative;
    box-shadow: 0 0 10px rgba(255, 85, 85, 0.8);
  }

  .led-body::after {
    content: '';
    position: absolute;
    top: 5px;
    left: 7px;
    width: 16px;
    height: 16px;
    background: rgba(255, 255, 255, 0.4);
    border-radius: 50%;
  }

  .led-pin {
    width: 2px;
    height: 40px;
    background-color: #333;
    position: relative;
  }

  .anode {
    height: 50px; /* Longer pin for the anode */
    margin-right: 15px;
    background-color: #666;
  }

  .cathode {
    height: 40px; /* Shorter pin for the cathode */
    margin-left: 15px;
    background-color: #333;
    position: absolute;
    margin-top: 45px;
  }

  canvas {
    border: 1px solid #ccc;
    display: block;
    margin: 10px auto;
  }
  #pwmCanvas {
    background-color: #fefefe;
  } 
</style>

In this section, we will explore what is PWM and why we need it.

## Digital vs Analog
In a digital circuit, signals are either high (such as 5V or 3.3V) or low (0V), with no in-between values. These two distinct states make digital signals ideal for computers and digital devices, as they're easy to store, read, and transmit without losing accuracy.   

Analog signals, however, can vary continuously within a range, allowing for any value between a High and Low voltage.  This smooth variation is valuable for applications requiring fine control, such as adjusting audio volume or light brightness.

Devices like servo motors and LEDs(for dimming effect) often need gradual, precise control over voltage, which analog signals provide through their continuous range.

Microcontrollers use PWM to bridge this gap.

## What is PWM?
PWM stands for **Pulse Width Modulation**, creates an analog-like signal by rapidly pulsing a digital signal on and off. The average output voltage, controlled by adjusting the pulse's high duration or "duty cycle," can simulate a continuous analog level. 
 
 <img style="display: block; margin: auto;" alt="LED PWM" src="../images/led-pwm.jpg" />

## Duty Cycle

The **duty cycle** is the percentage of time a signal is "on" during one complete cycle. 

For example:
- 100% duty cycle means the signal is always on.
- 50% duty cycle means the signal is on half the time and off half the time.
- 0% duty cycle means the signal is always off.


Here is the interactive simulation. Use the sliders to adjust the duty cycle and frequency, and watch how the pulse width and LED brightness change.  The upper part of the square wave represents when the signal is high (on). The lower part represents when the signal is low(off)

<canvas id="pwmCanvas" width="800" height="200"></canvas>
<div class="led-container">
  <div class="led-body" id="ledBody"></div>
  <div class="led-pin anode"></div>
  <div class="led-pin cathode"></div>
</div>

<div class="slider-container">
  <label for="dutyCycle">Duty Cycle (%): </label>
  <input type="range" id="dutyCycle" min="0" max="100" value="50">
  <span id="dutyCycleValue">50</span>%
</div>
<div class="slider-container">
  <label for="frequency">Frequency (Hz): </label>
  <input type="range" id="frequency" min="1" max="50" value="10">
  <!-- <span id="frequencyValue">x</span> Hz -->
</div>

If you change the duty cycle from "low to high" and "high to low" in the simulation, you should notice the LED kind of giving a dimming effect.

## Period and Frequency
Period is the total time for one on-off cycle to complete. 

The frequency of a PWM signal is the number of cycles it completes in one second, measured in Hertz (Hz).  Frequency is the inverse of the period.  So, a higher frequency means a shorter period, resulting in faster switching between HIGH and LOW states.

\\[
\text{Frequency (Hz)} = \\frac{1}{\text{Period (s)}}
\\]

So if the period is 1 second, then the frequency will be 1HZ.

\\[
1 \text{Hz} = \\frac{1 \text{ cycle}}{1 \text{ second}} = \\frac{1}{1 \text{ s}}
\\]

For example, if the period is 20ms(0.02s), the frequency will be 50Hz.

\\[
\text{Frequency} = \\frac{1}{20 \text{ ms}} = \\frac{1}{0.02 \text{ s}} = 50 \text{ Hz}
\\]


**Calculating Cycle count from Frequency per second**

The Formula to calculate cycle count:  
\\[
\text{Cycle Count} = \text{Frequency (Hz)} \\times \text{Total Time (seconds)}
\\]

If a PWM signal has a frequency of 50Hz, it means it completes 50 cycles in one second.


<script>
  const pwmCanvas = document.getElementById('pwmCanvas');
  const pwmCtx = pwmCanvas.getContext('2d');
  
  const dutyCycleSlider = document.getElementById('dutyCycle');
  const dutyCycleValue = document.getElementById('dutyCycleValue');
  const frequencySlider = document.getElementById('frequency');
  const frequencyValue = document.getElementById('frequencyValue');
  const ledBody = document.getElementById('ledBody');

  let dutyCycle = 50; // Initial duty cycle in percentage
  let frequency = 10; // Initial frequency in Hz

  function drawPWM() {
    pwmCtx.clearRect(0, 0, pwmCanvas.width, pwmCanvas.height);

    const period = 1000 / frequency; // Period in ms
    const onTime = period * (dutyCycle / 100); // On time in ms
    const offTime = period - onTime; // Off time in ms

    const totalWidth = pwmCanvas.width;
    const cycles = frequency; // Number of cycles to display
    const cycleWidth = totalWidth / cycles;

    pwmCtx.strokeStyle = 'black';
    pwmCtx.lineWidth = 2;
    pwmCtx.beginPath();

    let x = 0;

    if (dutyCycle === 100) {
      pwmCtx.moveTo(0, 50);
      pwmCtx.lineTo(pwmCanvas.width, 50);
    } else if (dutyCycle === 0) {
      pwmCtx.moveTo(0, 150);
      pwmCtx.lineTo(pwmCanvas.width, 150);
    } else {
      for (let i = 0; i < cycles; i++) {
        const highWidth = (onTime / period) * cycleWidth;
        const lowWidth = (offTime / period) * cycleWidth;

        pwmCtx.moveTo(x, 50);
        pwmCtx.lineTo(x + highWidth, 50);
        pwmCtx.lineTo(x + highWidth, 150);
        pwmCtx.lineTo(x + highWidth + lowWidth, 150);
        pwmCtx.lineTo(x + highWidth + lowWidth, 50);

        x += cycleWidth;
      }
    }
    pwmCtx.stroke();
  }

  function updateLED() {
    const brightness = dutyCycle / 100;
    
    ledBody.style.background = `radial-gradient(circle at center, rgba(255, 85, 85, ${brightness}), #cc0000)`;
  }

  function update() {
    dutyCycle = parseInt(dutyCycleSlider.value, 10);
    frequency = parseInt(frequencySlider.value, 10);

    dutyCycleValue.textContent = dutyCycle;
    // frequencyValue.textContent = frequency;

    drawPWM();
    updateLED();
  }

  dutyCycleSlider.addEventListener('input', update);
  frequencySlider.addEventListener('input', update);

  // Initial draw
  drawPWM();
  updateLED();
</script>
