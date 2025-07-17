# Delay


Just toggling the LED state between High and Low is not enough. If you do that without any delay, the LED will blink so fast that you won't even notice it. To create a visible blinking effect, we need to add a short pause between the toggles.

Add the necessary import:
```rust
use esp_hal::time::{Duration, Instant};
```

Here is how you can do it:

```rust

loop {
    led.toggle();
    blocking_delay(Duration::from_millis(500));
}
```

We define the delay function like this:

```rust

fn blocking_delay(duration: Duration) {
    let delay_start = Instant::now();
    while delay_start.elapsed() < duration {}
}
```

This blocking_delay function simply waits in a loop until the specified duration has passed. It uses Instant::now() to record the start time and then continuously checks how much time has elapsed. This is called a busy wait or blocking delay, because the CPU is stuck in the loop and can't do anything else during this time. While it's not the most efficient way, it's simple and works well for quick tests like blinking an LED.
