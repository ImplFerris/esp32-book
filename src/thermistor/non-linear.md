{{#title How Thermistor Non-Linearity Affects Temperature Measurement}}

# Thermistor Non-Linearity

Thermistors have a non-linear relationship between resistance and temperature, meaning that as the temperature changes, the resistance doesn't change in a straight-line pattern. The behavior of thermistors can be described using the Steinhart-Hart equation or the B equation. 

<img style="display: block; margin: auto;" alt="pico2" src="./images/thermistor-non-linearity.jpg"/>

The B equation is simple to calculate using the B value, which you can find in the thermistor's datasheet or from the manufacturer. On the other hand, the Steinhart-Hart equation uses A, B, and C coefficients. Some manufacturers provide these coefficients for their thermistors. When appropriate coefficients are available, the Steinhart-Hart equation can provide more accurate temperature calculations over a wider temperature range.


In the next chapters, we will see in detail how to use B equation and Steinhart-Hart equation to determine the temperature. 

### Referemce
- [The B parameter vs. Steinhart-Hart equation](https://blog.meteodrenthe.nl/2022/09/07/the-b-parameter-vs-steinhart-hart-equation/)
- [Characterising Thermistors – A Quick Primer, Beta Value & Steinhart-Hart Coefficients](https://community.element14.com/challenges-projects/design-challenges/experimenting-with-thermistors/b/challenge-blog/posts/blog-3-characterising-thermistors-a-quick-primer-beta-value-steinhart-hart-coefficients)
