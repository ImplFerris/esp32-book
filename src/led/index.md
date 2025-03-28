# LED Fading effect

In this section, we will learn how to create a fading effect(i.e. reducing and increasing the brightness) for an LED using the ESP32. First, we will fade the onboard LED, which is connected to GPIO 2. 

## Prerequisites
To make the fading effect, we use a technique called PWM (Pulse Width Modulation). 

Recommended to read these chapters
- Intro to the PWM section [here](../core-concepts/pwm/index.md).
- [LEDC](../core-concepts/pwm/led-pwm-controller.md)

We will gradually increment the PWM's duty cycle to increase the brightness, then we gradually decrement the PWM duty cycle to reduce the brightness of the LED. This effectively creates the fading effect on the LED

## The Eye

"
Come in close... Closer... 

Because the more you think you see... The easier it'll be to fool you... 

Because, what is seeing?.... You're looking but what you're really doing is filtering, interpreting, searching for meaning...
"

In PWM, we basically set a percentage to specify how long the signal stays HIGH (ON) and LOW (OFF) in each cycle. For instance, with a 50% duty cycle, the signal is ON for half the time and OFF for the other half, making the LED light up for just half the cycle.

Now, here's the cool part: when this switching happens super quickly, our eyes can't keep up. Instead of seeing the blinking, it just looks like the brightness changes! The longer the LED stays ON, the brighter it seems, and the shorter it's ON, the dimmer it looks. It's like tricking your brain into thinking the LED is smoothly dimming or brightening. PWM: the magician of electronics!
