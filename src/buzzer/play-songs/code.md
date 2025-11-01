## Code

### Generate project using esp-generate

You have done this step already in the quick start section. 

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 buzzer-song
```

This will open a screen asking you to select options.  In the latest esp-hal, ledc requires us to explicitly enable the unstable features. 

- So you select the option "Enable unstable HAL features."

Then save it by pressing "s" in the keyboard.

### Buzzer Pin

We will set GPIO 33 as our output pin. This is the pin where we connected the positive pin of the buzzer.

```rust
let mut buzzer = peripherals.GPIO33;
let ledc = Ledc::new(peripherals.LEDC);
```

### Song instance
Create instance of the Song struct with the tempo of the song we are going to play.
```rust
let song = Song::new(pink_panther::TEMPO);
```

### Playing Music Notes with PWM

We will loop through the song's notes, using each note's frequency and duration. We also add a 10% pause to each note's duration. The frequency constants are defined as f64, which we convert to u64 and use with the Hz. The timer and PWM channel configurations are as usual, with one difference: in the timer, we set the frequency to match the current note. We set the duty cycle to 50%.
 
```rust
 for (note, duration_type) in pink_panther::MELODY {
    // get music notes duration
    let note_duration = song.calc_note_duration(duration_type) as u64;
    let pause_duration = note_duration / 10; // 10% of note_duration
    if note == music::REST {
        blocking_delay(Duration::from_millis(note_duration));
        continue;
    }

    // Initialize LEDC
    let freq = Rate::from_hz(note as u32);
    let mut hstimer0 = ledc.timer::<HighSpeed>(timer::Number::Timer0);
    hstimer0
        .configure(timer::config::Config {
            duty: timer::config::Duty::Duty10Bit,
            clock_source: timer::HSClockSource::APBClk,
            frequency: freq,
        })
        .unwrap();

    // Initialize LEDC Channel with duty cycle 50%
    let mut channel0 = ledc.channel(channel::Number::Channel0, buzzer.reborrow());
    channel0
        .configure(channel::config::Config {
            timer: &hstimer0,
            duty_pct: 50,
            drive_mode: DriveMode::PushPull,
        })
        .unwrap();

    blocking_delay(Duration::from_millis(note_duration - pause_duration)); // play 90%
    channel0.set_duty(0).unwrap();
    blocking_delay(Duration::from_millis(pause_duration)); // Pause for 10%
}
```


## Clone the existing project
You can clone (or refer) project I created and navigate to the `buzzer-song` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/buzzer-song
```


## The Full code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use esp_hal::clock::CpuClock;
use esp_hal::main;
use esp_hal::time::{Duration, Instant};

use buzzer_song::{
    music::{self, Song},
    pink_panther,
};

// LEDC
use esp_hal::gpio::DriveMode;
use esp_hal::ledc::HighSpeed;
use esp_hal::ledc::timer;
use esp_hal::{
    ledc::{
        Ledc,
        channel::{self, ChannelIFace},
        timer::TimerIFace,
    },
    time::Rate,
};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[main]
fn main() -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    let mut buzzer = peripherals.GPIO33;

    let ledc = Ledc::new(peripherals.LEDC);

    let song = Song::new(pink_panther::TEMPO);

    for (note, duration_type) in pink_panther::MELODY {
        let note_duration = song.calc_note_duration(duration_type) as u64;
        let pause_duration = note_duration / 10; // 10% of note_duration
        if note == music::REST {
            blocking_delay(Duration::from_millis(note_duration));
            continue;
        }
        let freq = Rate::from_hz(note as u32);

        let mut hstimer0 = ledc.timer::<HighSpeed>(timer::Number::Timer0);
        hstimer0
            .configure(timer::config::Config {
                duty: timer::config::Duty::Duty10Bit,
                clock_source: timer::HSClockSource::APBClk,
                frequency: freq,
            })
            .unwrap();

        let mut channel0 = ledc.channel(channel::Number::Channel0, buzzer.reborrow());
        channel0
            .configure(channel::config::Config {
                timer: &hstimer0,
                duty_pct: 50,
                drive_mode: DriveMode::PushPull,
            })
            .unwrap();

        blocking_delay(Duration::from_millis(note_duration - pause_duration)); // play 90%

        channel0.set_duty(0).unwrap();
        blocking_delay(Duration::from_millis(pause_duration)); // Pause for 10%
    }

    loop {
        blocking_delay(Duration::from_millis(5));
    }
}

fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```

