# Circuit

We will use I2C for communication between the ESP32 and the OLED display. For I2C, two GPIO pins need to be configured as SDA (Serial Data) and SCL (Serial Clock). On the ESP32, we can use any GPIO pins for I2C. We'll configure GPIO pin 18 as SCL and GPIO pin 23 as SDA. The VCC pin of the OLED will be connected to the 3.3V pin of the ESP32, and the Ground pin will be connected to the ESP32's ground.


<table style="margin-bottom:20px">
  <thead>
    <tr>
      <th>ESP32 Pin</th>
      <th style="width: 250px; margin: 0 auto;">Wire</th>
      <th>OLED Pin</th>
      <th>Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>3.3V</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire red" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>VCC</td>
      <td>Supplies power to the OLED display.</td>
    </tr>
    <tr>
      <td>GND</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire black" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>Ground</td>
      <td>Connects to the ground pin of the OLED.</td>
    </tr>
    <tr>
      <td>GPIO 18</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire blue" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>SCL</td>
      <td>Connects the clock signal (SCL) for I2C communication.</td>
    </tr>
    <tr>
      <td>GPIO 23</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire yellow" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>SDA </td>
      <td>Connects the data signal (SDA) for I2C communication.</td>
    </tr>
  </tbody>
</table>

<img style="display: block; margin: auto;" title="esp32 oled circuit" src="./images/connecting esp32 with oled ssd1306 circuit.png"/>
