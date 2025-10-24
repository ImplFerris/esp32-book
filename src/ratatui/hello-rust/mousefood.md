# Hello, Rat

In this chapter, we'll write a simple program to get started with Ratatui. We won't do anything fancy here, just the basics to see how it works.

## Prerequisites

- You'll need a TFT Display for this chapter. If you haven't completed the [TFT Display chapter](./tft-display/index.md), I recommend finishing that first and then coming back here. Since the circuit connections between the TFT Display and ESP32 are already explained there, we won't repeat those instructions here.​

- Ratatui has a nice [tutorial to get started](https://ratatui.rs/tutorials/hello-ratatui/) for building TUI apps. If you already know Ratatui, this exercise is pretty simple - just gluing things together. If you're new, check out the official Ratatui tutorials later to get familiar with the basics and build better UIs.


## Generate project using esp-generate

The ratatui requires

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 webserver-html
```

This will open a screen asking you to select options. In order to Enable Wi-Fi, we will first need to enable "unstable" and "alloc" features. If you noticed, until you select these two options, you wont be able to enable Wi-Fi option. So select one by one

- First, select the option "Enable unstable HAL features."
- Select the option "Enable allocations via the esp-alloc crate."
- Now, you can enable "Enable Wi-Fi via esp-wifi crate."
- Select the option "Adds embassy framework support".

Enable the logging feature also

- So, scroll to "Flashing, logging and debugging (espflash)" and hit Enter.
- Then, Select "Use defmt to print messages".

Just save it by pressing "s" in the keyboard.
