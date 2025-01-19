## Connecting RC522 with ESP32 
 
## Pinout diagram of RC522
There are 8 pins in the RC522 RFID module.
<a href="./images/rc522-pinout.jpg"><img style="display: block; margin: auto;" alt="pinout diagram of RC522" src="./images/rc522-pinout.jpg"/></a>

<table style="border-collapse: collapse; width: 100%; border: 1px solid black;">
  <tr style="border: 1px solid black;">
    <th style="background-color: #009B77; border: 1px solid black;">Pin</th>
    <th style="border: 1px solid black;">SPI Function</th>
    <th style="border: 1px solid black;">I²C Function</th>
    <th style="border: 1px solid black;">UART Function</th>
    <th style="border: 1px solid black;">Description</th>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #ff0000; color: white; border: 1px solid black;">3.3V</td>
    <td style="border: 1px solid black;">Power</td>
    <td style="border: 1px solid black;">Power</td>
    <td style="border: 1px solid black;">Power</td>
    <td style="border: 1px solid black;">Power supply (3.3V).</td>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #000; color: white; border: 1px solid black;">GND</td>
    <td style="border: 1px solid black;">Ground</td>
    <td style="border: 1px solid black;">Ground</td>
    <td style="border: 1px solid black;">Ground</td>
    <td style="border: 1px solid black;">Ground connection.</td>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #7B3F00; color: white; border: 1px solid black;">RST</td>
    <td style="border: 1px solid black;">Reset</td>
    <td style="border: 1px solid black;">Reset</td>
    <td style="border: 1px solid black;">Reset</td>
    <td style="border: 1px solid black;">Reset the module.</td>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #EFEFEF; border: 1px solid black;color:black;">IRQ</td>
    <td style="border: 1px solid black;">Interrupt (optional)</td>
    <td style="border: 1px solid black;">Interrupt (optional)</td>
    <td style="border: 1px solid black;">Interrupt (optional)</td>
    <td style="border: 1px solid black;">Interrupt Request (IRQ) informs the microcontroller when an RFID tag is detected. Without using IRQ, the microcontroller would need to constantly poll the module.</td>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #FFC000; color: black; border: 1px solid black;">MISO</td>
    <td style="border: 1px solid black;">Master-In-Slave-Out</td>
    <td style="border: 1px solid black;">SCL</td>
    <td style="border: 1px solid black;">TX</td>
    <td style="border: 1px solid black;">In SPI mode, it acts as Master-In-Slave-Out (MISO). In I²C mode, it functions as the clock line (SCL). In UART mode, it acts as the transmit pin (TX).</td>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #008000; color: white; border: 1px solid black;">MOSI</td>
    <td style="border: 1px solid black;">Master-Out-Slave-In</td>
    <td style="border: 1px solid black;">-</td>
    <td style="border: 1px solid black;">-</td>
    <td style="border: 1px solid black;">In SPI mode, it acts as Master-Out-Slave-In (MOSI).</td>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #0F52BA; color: white; border: 1px solid black;">SCK</td>
    <td style="border: 1px solid black;">Serial Clock</td>
    <td style="border: 1px solid black;">-</td>
    <td style="border: 1px solid black;">-</td>
    <td style="border: 1px solid black;">In SPI mode, it acts as the clock line that synchronizes data transfer.</td>
  </tr>
  <tr style="border: 1px solid black;">
    <td style="background-color: #FF5F1F; color: white; border: 1px solid black;">SDA</td>
    <td style="border: 1px solid black;">CS (or SS)</td>
    <td style="border: 1px solid black;">SDA</td>
    <td style="border: 1px solid black;">RX</td>
    <td style="border: 1px solid black;">In SPI mode, it acts as the Chip select (CS, also referred as Slave Select). In I²C mode, it serves as the data line (SDA). In UART mode, it acts as the receive pin (RX).</td>
  </tr>
</table>

## Connecting the RFID Reader to the ESP32
To establish communication between the ESP32 and the RFID Reader, we will use the SPI (Serial Peripheral Interface) protocol.  The SPI interface can handle data speed up to 10 Mbit/s. We wont be utilizing the following Pins: RST, IRQ at the moment.

<table>
  <thead>
    <tr>
      <th>ESP32 Pin</th>
      <th style="width: 250px; margin: 0 auto;">Wire</th>
      <th>RFID Reader Pin</th>
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
      <td>3.3V</td>
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
    <tr>
      <td>GPIO 5</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire green" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>Labeled as SDA, it acts as the CS pin when using SPI.</td>
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
      <td>GPIO 23</td>
      <td style="text-align: center; vertical-align: middle; padding: 0;">
        <div class="wire orange" style="width: 200px; margin: 0 auto;">
          <div class="male-left"></div>
          <div class="male-right"></div>
        </div>
      </td>
      <td>MOSI</td>
    </tr>
  </tbody>
</table>

<img style="display: block; margin: auto;" alt="connecting RC522 with ESP32" src="./images/connecting-esp32-with-rfid-rc522.png"/>
