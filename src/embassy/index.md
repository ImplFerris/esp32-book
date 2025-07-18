## Async

Async programming allows tasks to run concurrently without blocking each other.  In embedded systems, it enables microcontrollers to handle multiple tasks, such as reading sensors or controlling other peripherals without waiting for each task to finish. You can read the "[Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/intro.html)" for more details.

## Embassy


So far, we have worked with code that runs in blocking mode. This means that whenever we ask the program to do something like `delay` for a while or wait for a button press, the CPU stops and waits until that task is finished before continuing. This is simple to understand and works well for small programs, but it becomes limiting when we want to handle multiple tasks at the same time, like reading a sensor, and listening for input; all without blocking each other.

This is where [Embassy](https://github.com/embassy-rs/embassy) comes in. Embassy is an async runtime designed for embedded systems. It allows us to write non-blocking code using Rust's async and await features. Instead of waiting and wasting CPU time, tasks can pause and let others run, making better use of the processor and enabling more responsive and power-efficient applications. It can be used it with ESP32, Pico and other microcontrollers. 

For example, with Embassy, we can blink an LED and listen for a touch or button input at the same time, without writing complex interrupt-based code manually.

## HALs
Embassy provides async-ready Hardware Abstraction Layers (HALs) for several microcontroller families, offering safe and idiomatic Rust APIs so you can interact with hardware without dealing with low-level registers. 

Official HALs include embassy-stm32 (STM32), embassy-nrf (nRF52/53/54/91), embassy-rp (RP2040), and embassy-mspm0 (TI MSPM0). Embassy also works with community HALs like **esp-hal (ESP32)**, ch32-hal (CH32V), mpfs-hal (PolarFire), and py32-hal (Puya PY32), making it easy to write portable, async code across many platforms.


## Batteries included

Embassy comes with many built-in features that make embedded development easier. For example, it includes embassy-time for handling timers and delays, embassy-net for networking support, and embassy-usb for building USB device functionality and much more.


## Useful Resources

- [Embassy Book](https://embassy.dev/book/#_introduction) : The Embassy Book is for everyone who wants to use Embassy and understand how Embassy works.
- [Embassy Github](https://github.com/embassy-rs/embassy)
