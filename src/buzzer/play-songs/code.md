## Code

### Generate project using esp-generate

You have done this step already in the quick start section. 

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 active-buzzer
```

This will open a screen asking you to select options. For now, we dont need to select any options. Just save it by pressing "s" in the keyboard.

### Buzzer Pin

We will set GPIO 33 as our output pin with an initial Low state. This is the pin where we connected the positive pin of the buzzer.

```rust
let mut buzzer = Output::new(peripherals.GPIO33, Level::Low);
```

### Song instance
Create instance of the Song struct with the tempo of the song we are going to play.
```rust
    let song = Song::new(pink_panther::TEMPO);
```

### Playing Music Notes with PWM

We will loop through the song's notes, using each note's frequency and duration. We also add a 10% pause to each note's duration. The frequency constants are defined as f64, which we convert to u32 and use with the Hz trait. The timer and PWM channel configurations are as usual, with one difference: in the timer, we set the frequency to match the current note. We set the duty cycle to 50%.
 
```rust
for (note, duration_type) in pink_panther::MELODY {
    let note_duration = song.calc_note_duration(duration_type);
    let pause_duration = note_duration / 10; // 10% of note_duration
    if note == music::REST {
        delay.delay_millis(note_duration);
        continue;
    }
    let freq = (note as u32).Hz();

    let mut hstimer0 = ledc.timer::<HighSpeed>(timer::Number::Timer0);
    hstimer0
        .configure(timer::config::Config {
            duty: timer::config::Duty::Duty10Bit,
            clock_source: timer::HSClockSource::APBClk,
            frequency: freq,
        })
        .unwrap();

    let mut channel0 = ledc.channel(channel::Number::Channel0, &mut buzzer);
    channel0
        .configure(channel::config::Config {
            timer: &hstimer0,
            duty_pct: 50,
            pin_config: channel::config::PinConfig::PushPull,
        })
        .unwrap();

    delay.delay_millis(note_duration - pause_duration); // play 90%
    channel0.set_duty(0).unwrap();
    delay.delay_millis(pause_duration); // Pause for 10%
}
```


## Clone the existing project
You can clone (or refer) project I created and navigate to the `buzzer-song` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/buzzer-song
```
