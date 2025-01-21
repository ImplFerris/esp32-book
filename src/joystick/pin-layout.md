# Pin layout

The joystick has a total of 5 pins: power supply, ground, X-axis output, Y-axis output, and switch output pin.

<img style="display: block; margin: auto;width:400px;margin-bottom: 10px;" alt="joystick" src="./images/joystick-pin-layout.jpg"/>

<table border="1" style="border-collapse: collapse; width: 100%;">
  <thead>
    <tr>
      <th style="width:14%">Joystick Pin</th>
      <th>Details</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><span class="slanted-text black">GND</span></td>
      <td>Ground pin. Should be connected to the Ground of the circuit.</td>
    </tr>
    <tr>
      <td><span class="slanted-text red">VCC</span></td>
      <td>Power supply pin (typically 5V or 3.3V ).</td>
    </tr>
    <tr>
      <td><span class="slanted-text green">VRX</span></td>
      <td>The X-axis analog output pin varies its voltage based on the joystick's horizontal position, ranging from 0V to VCC as the joystick is moved left and right.</td>
    </tr>
    <tr>
      <td><span class="slanted-text blue">VRY</span></td>
      <td>The Y-axis analog output pin varies its voltage based on the joystick's vertical position, ranging from 0V to VCC as the joystick is moved up and down.</td>
    </tr>
    <tr>
      <td><span class="slanted-text purple">SW</span></td>
      <td>Switch pin. When the joystick knob is pressed, this pin is typically pulled LOW (to GND).</td>
    </tr>
  </tbody>
</table>

