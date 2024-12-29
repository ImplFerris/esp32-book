## Creating a Beep Sound with an Active Buzzer Using ESP32 and Rust

Since you already know that an active buzzer is simple to use, you can make it beep just by powering it. In this exercise, we'll make it beep with just a little code.


### Hardware Requirements
- **Active Buzzer**
- **Female-to-Male** or **Male-to-Male** jumper wires (depending on your setup)

### Generate project using esp-generate

You have done this step already in the quick start section. 

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 active-buzzer
```

This will open a screen asking you to select options. For now, we dont need to select any options. Just save it by pressing "s" in the keyboard.

## Code

We will set GPIO 33 as our output pin with an initial Low state. This is the pin where we connected the positive pin of the buzzer.

```rust
let mut buzzer = Output::new(peripherals.GPIO33, Level::Low);
```

The logic is straightforward: set the buzzer pin to High for 500 milliseconds, then to Low for another 500 milliseconds in a loop. This causes the buzzer to produce a beeping sound.

```rust
    let delay = Delay::new();
    loop {
        buzzer.set_high();
        delay.delay_millis(500);
        buzzer.set_low();
        delay.delay_millis(500);
    }
```

## Clone the existing project
You can clone (or refer) project I created and navigate to the `active-buzzer` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/active-buzzer
```

