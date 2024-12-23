# LED PWM Controller(LEDC)

The ESP32 has LED PWM Controller(LEDC) that generates PWM signals for controlling LEDs(example, dimming effect). However, its functionality isn't limited to LEDs;you can use it for other applications as well.  If you are not familiar with PWM(Pulse Width Modulation), i recommend you to check the PWM section [here](../core-concepts/pwm.md). 

The LEDC includes 16 independent PWM generators and supports a maximum PWM duty cycle resolution of 20 bits. The 16 PWM channels further classified into two types: 8 high speed channel and 8 low speed channels.

<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">High vs Low Speed channels</b>
        <p>High-speed channels use hardware to automatically adjust the PWM duty cycle in a glitch-free manner, ensuring smooth operation. In contrast, low-speed channels rely on software to manually adjust the duty cycle.</p>
    </div>
</div>

The PWM controller can automatically increase or decrease the duty cycle gradually, allowing for smooth fades without using the processor. For more details, refer to page 390 of the [ESP32 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf#ledpwm).

