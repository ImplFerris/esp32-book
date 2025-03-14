# Playing Songs on a Passive Buzzer Using Rust and ESP32

We are going to play songs on the buzzer.  If you're unsure about musical notes and sheet music, feel free to check out the quick theory I've provideded [here](./music-theory.md).

I've splitted the code into rust module(you can do it in single file as we have done so far): [`music`](./music-module.md), [`got`](./got-module.md).

A passive buzzer is recommended for this exercise, though you can use either a passive or active buzzer.

## PWM
We will use PWM to adjust the frequency of the signal sent to the buzzer, with each frequency corresponding to a musical note. The frequency (musical note) will be held for a specific duration before switching to the next note, as per the music sheet.

For example, the note "A4" is 440Hz and is played for some X duration. We set this frequency in PWM and add a delay for that X duration.

I recommend reading the [PWM section](../../core-concepts/pwm/index.md) to familiarize yourself with how PWM works.


## Song Repository

We will playing the Pink Panther theme in this exercise. However, you can refer to the rust-embedded-songs [repository](https://github.com/ImplFerris/rust-embedded-songs/) i created and use different songs also.


## lib.rs submodules
Update lib.rs to define the submodule, then create the `music.rs` and `pink_panther.rs` files.

```rust
#![no_std]
pub mod music;
pub mod pink_panther;
```
