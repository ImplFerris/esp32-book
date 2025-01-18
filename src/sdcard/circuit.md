## Circuit

### microSD Card Pin Mapping for SPI Mode

We'll focus only on the microSD card since that's what we're using. The microSD has 8 pins, but we only need 6 for SPI mode. You may have noticed that the SD card reader module we have also has only 6 pins, with markings for the SPI functions. The table below shows the microSD card pins and their corresponding SPI functions.
 
<div style="display: flex; align-items: center;gap:18px;">
  <img style="width: 180px;" alt="microSD Card Pin Diagram" src="./images/micro-sd-card-pin.png"/>
  <table>
    <thead>
      <tr>
        <th>microSD Card Pin</th>
        <th>SPI Function</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>1</td>
        <td>-</td>
      </tr>
      <tr>
        <td>2</td>
        <td>Chip Select (CS); also referred as Card Select</td>
      </tr>
      <tr>
        <td>3</td>
        <td>Data Input (DI) - corresponds to MOSI. To receive data from the microcontroller.</td>
      </tr>
      <tr>
        <td>4</td>
        <td>VDD - Power supply (3.3V)</td>
      </tr>
      <tr>
        <td>5</td>
        <td>Serial Clock (SCK)</td>
      </tr>
      <tr>
        <td>6</td>
        <td>Ground (GND)</td>
      </tr>
      <tr>
        <td>7</td>
        <td>Data Output (DO) - corresponds to MISO. To send data from the microSD card to the microcontroller.</td>
      </tr>
      <tr>
        <td>8</td>
        <td>-</td>
      </tr>
    </tbody>
  </table>
</div>

### Connecting the ESP32 to the SD Card Reader

The microSD card operates at 3.3V, so using 5V to power it could damage the card. However, the reader module comes with an onboard voltage regulator and logic shifter, allowing it to safely be connected to the 5V power supply of the ESP32.
 
<table>
  <thead>
    <tr>
      <th>ESP32 Pin</th>
      <th style="width: 250px; margin: 0 auto;">Wire</th>
      <th>SD Card Pin</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>GPIO 5</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire green" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>CS</td>
    </tr>
    <tr>
      <td>GPIO 18</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire blue" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>SCK</td>
    </tr>
    <tr>
      <td>GPIO 23</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire orange" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>MOSI</td>
    </tr>
    <tr>
      <td>GPIO 19</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire yellow" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>MISO</td>
    </tr>
        <tr>
      <td>Vin</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire red" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>VCC</td>
    </tr>
    <tr>
      <td>GND</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire black" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>GND</td>
    </tr>
  </tbody>
</table>
<br/>
<img style="display: block; margin: auto;" alt="SD Card reader pico connection" src="./images/connecting-micro-sdcard-reader-module-with-esp32-devkit-v1.png"/>
