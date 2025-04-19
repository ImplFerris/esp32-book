# Dashboard

In this chapter, we'll build the main logic. The idea is to update the dashboard every so often;say, every 10 minutes. To do that, we'll fetch the latest weather data by calling the access_website function from the weather module, and then update each part of the dashboard with the new info.

## Type aliases
We'll create type aliases for the SPI device and the e-paper display to make the code easier to read and work with.

```rust
type SpiDevice = ExclusiveDevice<Spi<'static, esp_hal::Blocking>, Output<'static>, Delay>;
type EPD = Epd1in54<SpiDevice, Input<'static>, Output<'static>, Output<'static>, Delay>;
```

## Dashboard struct

We'll define a Dashboard struct that takes the Wi-Fi stack, e-paper driver, and SPI device as inputs. The new function will initialize the struct and set up a default e-paper display.

```rust
pub struct Dashboard {
    display: Display1in54,
    wifi: Stack<'static>,
    epd: EPD,
    spi_dev: SpiDevice,
}

impl Dashboard {
    pub fn new(wifi: Stack<'static>, epd: EPD, spi_dev: SpiDevice) -> Self {
        Self {
            display: Display1in54::default(),
            wifi,
            epd,
            spi_dev,
        }
    }
}
```

Note: The following functions which take self as the first parameter should also be included inside the impl block.

## Dashboard Startup

This will be the starting function, which we'll call next in the main.rs file. It will accept SHA and RSA peripherals as input, which are needed to set up TLS. We will also instantiate the WeatherApi.

I rotated the display 90 degrees. Technically, you don't need to do this since the display we're using is 200x200 pixels. But if you're using a rectangular display, it makes more sense to rotate it.

In a loop, every 10 minutes, we'll call the refresh function which will fetch the latest data and update the display.

```rust
pub async fn start(&mut self, sha: SHA, rsa: RSA) {
    self.display.set_rotation(DisplayRotation::Rotate90);

    let tls = Tls::new(sha)
        .expect("TLS::new with peripherals.SHA failed")
        .with_hardware_rsa(rsa);

    let api = WeatherApi::new(self.wifi);
    loop {
        self.refresh(&api, tls.reference()).await;

        Timer::after(Duration::from_secs(60 * 10)).await;
    }
}
```

## Refreshing the Display with Latest Weather Data

First, we get the latest weather data by calling the `access_website` function from `WeatherApi`. Then, we wake up the e-paper display, since we will be putting it to sleep at the end. After that, we clear the previous frame and fill the display with white.

Next, we draw the updated weather info - first the date, then the weather icon and temperature. After that, we add humidity, wind speed, and finally the signature at the bottom (just a small "implRust" text). Once everything is drawn, we update the display with the new frame, wait for 5 seconds, and then put it to sleep.

```rust
pub async fn refresh(&mut self, api: &WeatherApi, tls_reference: TlsReference<'_>) {
    info!("Getting weather data");
    let weather_data = api.access_website(tls_reference).await;
    info!("Got weather data");

    self.epd.wake_up(&mut self.spi_dev, &mut Delay).unwrap();
    Timer::after(Duration::from_secs(5)).await;

    // Clear any existing image
    self.epd.clear_frame(&mut self.spi_dev, &mut Delay).unwrap();
    self.display.clear(Color::White).unwrap();
    self.epd
        .update_and_display_frame(&mut self.spi_dev, self.display.buffer(), &mut Delay)
        .unwrap();
    Timer::after(Duration::from_secs(5)).await;

    self.draw_date(weather_data.dt);

    self.draw_icon(weather_data.weather[0].id.icon(), Point::new(20, 50));
    self.draw_temperature(weather_data.main.temp, Point::new(20 + 70, 60));

    self.draw_humidity(weather_data.main.humidity);
    self.draw_wind(weather_data.wind.speed);

    self.draw_signature();

    self.epd
        .update_and_display_frame(&mut self.spi_dev, self.display.buffer(), &mut Delay)
        .unwrap();
    Timer::after(Duration::from_secs(5)).await;

    self.epd.sleep(&mut self.spi_dev, &mut Delay).unwrap();
}
```
## Get Icon helper function
This simple helper function takes the icon's name and the position where it should be drawn.  It maps the icon name to the corresponding image bytes, then uses tinybmp and embedded_graphics to convert the bytes into an image, which is then rendered on the display.

```rust
fn draw_icon(&mut self, icon_name: &'static str, pos: Point) {
    let img_bytes = self.get_icon(icon_name).unwrap();

    let bmp = Bmp::from_slice(img_bytes).unwrap();
    let image = Image::new(&bmp, pos);
    image.draw(&mut self.display).unwrap();
}

pub fn get_icon(&self, icon_name: &'static str) -> Option<&'static [u8]> {
    ICONS
        .iter()
        .find(|(name, _)| *name == icon_name)
        .map(|(_, img_bytes)| *img_bytes)
}
```

## Display Temperature
We will display the temperature on the screen, format it with a "°C" suffix, and draw a horizontal line below it.

```rust
fn draw_temperature(&mut self, temperature: f64, pos: Point) {
    let text_style = MonoTextStyle::new(&PROFONT_24_POINT, Color::Black);

    info!("Drawing temperature");
    let mut text: String<20> = String::new();
    write!(&mut text, "{}°C", temperature).unwrap();

    Text::with_baseline(&text, pos, text_style, Baseline::Top)
        .draw(&mut self.display)
        .unwrap();

    Line::new(Point::new(0, 105), Point::new(200, 105))
        .into_styled(PrimitiveStyle::with_stroke(Color::Black, 5))
        .draw(&mut self.display)
        .unwrap();
}
```

## Display Humidity

We will show the humidity icon on the screen, then display the humidity value next to it, followed by a vertical line for separation.

```rust
fn draw_humidity(&mut self, humidity: i32) {
    self.draw_icon("humidity_percentage.bmp", Point::new(5, 110));

    let text_style = MonoTextStyle::new(&PROFONT_18_POINT, Color::Black);

    let mut text: String<10> = String::new();
    write!(&mut text, "{}", humidity).unwrap();

    Text::with_baseline(&text, Point::new(5 + 50, 120), text_style, Baseline::Top)
        .draw(&mut self.display)
        .unwrap();

    Line::new(Point::new(5 + 85, 120), Point::new(5 + 85, 120 + 30))
        .into_styled(PrimitiveStyle::with_stroke(Color::Black, 5))
        .draw(&mut self.display)
        .unwrap();
}
```

## Draw Wind

We will display the wind speed on the screen by first drawing the wind icon. Then, we will show the wind speed value followed by the unit "m/s" on the display.

```rust
fn draw_wind(&mut self, wind_speed: f64) {
    self.draw_icon("air.bmp", Point::new(100, 110));

    let text_style = MonoTextStyle::new(&PROFONT_18_POINT, Color::Black);

    let mut text: String<10> = String::new();
    write!(&mut text, "{}", wind_speed).unwrap();

    Text::with_baseline(&text, Point::new(100 + 50, 120), text_style, Baseline::Top)
        .draw(&mut self.display)
        .unwrap();

    let text_style = MonoTextStyleBuilder::new()
        .font(&FONT_10X20)
        .text_color(Color::Black)
        .build();

    Text::with_baseline("m/s", Point::new(100 + 50, 140), text_style, Baseline::Top)
        .draw(&mut self.display)
        .unwrap();
}
```
## Display Date

We will display the current date on the screen by formatting it with the day, month, and year. Then, we will render the text at the specified position and draw a horizontal line below it for separation.

```rust
fn draw_date(&mut self, dt: DateTime<Utc>) {
    let text_style = MonoTextStyle::new(&PROFONT_24_POINT, Color::Black);

    let mut text: String<24> = String::new();
    write!(
        &mut text,
        "{} {} {}",
        dt.day(),
        month_name(dt.month()),
        dt.year()
    )
    .unwrap();

    Text::with_baseline(&text, Point::new(20, 10), text_style, Baseline::Top)
        .draw(&mut self.display)
        .unwrap();

    Line::new(Point::new(0, 45), Point::new(200, 45))
        .into_styled(PrimitiveStyle::with_stroke(Color::Black, 5))
        .draw(&mut self.display)
        .unwrap();
}
```

## Display Signature

This isn't related to the weather; we're just displaying "implRust" for fun. We calculate the center position, draw a black rectangle in the center, and place the text right in the middle.

```rust
fn draw_signature(&mut self) {
    let display_width = epd1in54_v2::WIDTH as i32;
    let rect_padding = 20;

    let rect_width = display_width - 2 * rect_padding;
    let rect_height = 40;
    let rect_x = rect_padding;
    let rect_y = 170;

    let style = PrimitiveStyleBuilder::new()
        .stroke_color(Color::Black)
        .stroke_width(3)
        .fill_color(Color::Black)
        .build();

    Rectangle::new(
        Point::new(rect_x, rect_y),
        Size::new(rect_width as u32, rect_height as u32),
    )
    .into_styled(style)
    .draw(&mut self.display)
    .unwrap();

    let text = "implRust";
    let text_style = MonoTextStyle::new(&PROFONT_24_POINT, Color::White);

    let char_width = PROFONT_24_POINT.character_size.width as i32;
    let text_width = text.len() as i32 * char_width;
    let text_x = rect_x + (rect_width - text_width) / 2;

    Text::with_baseline(
        text,
        Point::new(text_x as i32, rect_y),
        text_style,
        Baseline::Top,
    )
    .draw(&mut self.display)
    .unwrap();
}
```

## Month Name Helper Function
We have created a helper function that returns the abbreviated month name based on the month number.

```rust
fn month_name(month: u32) -> &'static str {
    match month {
        1 => "Jan",
        2 => "Feb",
        3 => "Mar",
        4 => "Apr",
        5 => "May",
        6 => "Jun",
        7 => "Jul",
        8 => "Aug",
        9 => "Sep",
        10 => "Oct",
        11 => "Nov",
        12 => "Dec",
        _ => "Err",
    }
}
```
