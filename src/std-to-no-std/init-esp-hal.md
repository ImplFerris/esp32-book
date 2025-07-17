# Initialize ESP HAL

We were able to successfully prepare a binary for the ESP32. However, the program does absolutely nothing at this stage. Let's bring it to life by replicating the same blinky program we saw in the Quick Start section.

Start by adding this import:

```rust
use esp_hal::clock::CpuClock;
```

Now, at the beginning of your main function, add this line:

```rust
let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
```

This line creates a default configuration for the HAL and sets the CPU clock to its maximum frequency. You can think of it as giving the chip some instructions on how fast it should run. We are telling it to run at its maximum speed. 

## Initialize Peripherals

Next, we initialize the peripherals using the HAL:
```rust
let peripherals = esp_hal::init(config);
```

This function sets up the system by configuring things like the CPU clock and the watchdog timer. After that, it gives us access to the chip's peripherals and clocks so we can start using them in our code.
