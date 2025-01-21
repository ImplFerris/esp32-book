# Project Ideas

## 1. Access Control with RFID and OLED

Build an access control system that displays "Access Granted" or "Access Denied" on an OLED display when RFID tags are scanned. Optionally, add a buzzer to provide audio feedback based on the access status.

### Components
- MFRC522 (RFID reader module) & RFID Tags/Cards
- 0.96-inch OLED Display (I2C interface)
- \[Optional\] Buzzer (for audio feedback)
- Power Supply (e.g., 5V adapter or battery)

### Related resources to learn
- [OLED Display](../oled/index.md)
- [Buzzer](../buzzer/index.md)

## 2. Automatic Garage door using RFID

I found an interesting video that demonstrates an automatic garage door system using RFID with Arduino. You can replicate this project using an ESP32 and Rust. [Watch the video here](https://www.youtube.com/watch?v=ICnYGbvkrpo).

### Components
- MFRC522 (RFID reader module) & RFID Tags/Cards
- Servo motor
- \[Optional\] Buzzer (optional, for audio feedback)
- Power Supply (e.g., 5V adapter or battery)


### Related resources to learn
- [Servo motor](../servo/index.md)



## 3. Simple Smart Door Lock System with ESP32 and RFID

A door model can be built using a cardboard box, where a servo motor opens the door when the correct key is presented. If the wrong key is used, the door remains closed, and optionally, a buzzer sounds for feedback. You can replicate this project using an ESP32 and Rust. [Watch the video here](https://www.youtube.com/watch?v=3xb2PLFjJxk).

### Components
- MFRC522 (RFID reader module) & RFID Tags/Cards
- Servo motor
- \[Optional\] Buzzer (optional, for audio feedback)
- Power Supply (e.g., 5V adapter or battery)


### Related resources to learn
- [Servo motor](../servo/index.md)


### Showcase
If you create a project based on this idea, feel free to send a pull request with your project link, and we'll feature it here!

- [ESP32 RFID Access Control](https://github.com/implferris/esp32-rfid-access): Smart Door Lock Simulation with Rust and ESP32, using RFID, optional servo motor, and OLED display to simulate and control door access.
