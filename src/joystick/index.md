# Joystick

In this section, we'll explore how to use the Joystick Module. It is similar to the joysticks found on PS2 (PlayStation 2) controllers. They are commonly used in gaming, as well as for controlling drones, remote-controlled cars, robots, and other devices to adjust position or direction.
 
## Meet the hardware - Joystick module

<img style="display: block; margin: auto;width:250px;" alt="joystick" src="./images/joystick.jpg"/>

You can move the joystick knob vertically and horizontally, it will send its position (X and Y axes) to the MCU (e.g., ESP32). Additionally, the knob can be pressed down like a button.  The joystick typically operates at 5V, but it can also be connected to 3.3V.

## How it works?
The joystick module has two 10K potentiometers: one for the X-axis and another for the Y-axis. It also includes a push button, which is visible.

When you move the joystick from right to left or left to right(X axis), you can observe one of the potentiometers moving accordingly. Similarly, when you move it up and down(Y-axis), you can observe the other potentiometer moving along.

<img style="display: block; margin: auto;width:550px;" alt="joystick" src="./images/joystick-potentiometers-push-button.jpg"/>

You can also observe the push-button being pressed when you press down on the knob.
 