# FAQ


### 1. Can I use other ESP32 dev boards?

Yes, you can use the code on most dev boards with an "ESP32" chip. However, pinouts may differ, so you might need to make some adjustments. Other than that, everything should work fine!

### 2. Can I use other ESP32 family chips like ESP32-C3?  
Yes, but the code may not work as is. ESP32 variants have different configurations-some may lack Bluetooth or Wi-Fi, while others have different specs. While some code may work across ESP32 variants, differences in architecture, peripherals, and GPIO mapping might require modifications.  

You can still read the concepts and try applying them to your chip. For adjustments, refer to the official ESP-HAL examples:  
[ESP-HAL Examples](https://github.com/esp-rs/esp-hal/tree/main/examples)

That said, I highly recommend getting an **ESP32 Devkit V1** board for a smoother experience. It's cheap and works well with the examples in this book. Later, you can experiment with other ESP32 variants!

### 3. I cloned the project, but it's giving errors. What should I do?  

If you cloned the example projects from the book and encountered errors, it's likely due to a Rust toolchain version mismatch. Since these projects use a **nightly** Rust version, breaking changes can occur over time.  

#### What can you do?  
You have two options:  

1. **Generate a new project with `esp-generate`**  as mentioned in the exercies
   - You can then refer to the book's code and tweak it if needed.  

2. **Downgrade your Rust toolchain to match the book**  
   - You can check the nightly version used in this book's projects [here](./dev-env.md#toolchains-for-risc-v-and-xtensa-targets).  
   - Install the matching version to ensure compatibility.  

### 4. I am getting import error while following the instructions?

Your code editor should typically assist with imports. However, in some cases, it might not work as expected. You can always cross-check your code with the "full code" section or clone the project to compare and add any missing imports.

### 5. Where can I find the documentation for esp-hal?
You can find the official documentation for esp-hal at the following link:

👉 [ESP-HAL Documentation](https://docs.espressif.com/projects/rust/esp-hal/1.0.0-beta.0/)

