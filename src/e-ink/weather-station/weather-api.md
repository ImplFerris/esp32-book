# Weather API

In this chapter, we'll learn how to send a request to the API and deserialize the JSON response into a struct.

I hope you have obtained API key for the openweathermap website. We can pass it through environment variable. So we define constant like this

```rust
const API_KEY: &str = env!("API_KEY");
```

## JSON to Struct

I want you to go through the API documentation at "https://openweathermap.org/current" and familiarize yourself with the JSON response structure. At the top level, I defined a WeatherData struct with fields like weather, main, and wind, each using custom struct types. To avoid cluttering the tutorial, I'll include the definitions of other structs at the end of the chapter.


```rust
#[derive(Debug, Deserialize)]
pub struct WeatherData {
    pub weather: Vec<Weather, 4>,
    pub main: Main,
    pub wind: Wind,
    #[serde(with = "chrono::serde::ts_seconds")]
    pub dt: DateTime<Utc>,
    pub name: String<20>,
}
```
Here, I'm only deserializing the fields we need and ignoring the rest of the JSON data.

## TLS Certificate

The OpenWeatherMap website does not support TLS 1.3. If it did, this task would have been much simpler;we could have just used the embedded-tls crate, which supports only TLS 1.3. embedded-tls also provides a TLSVerify::None option, allowing us to skip SSL certificate verification. Unfortunately, since we're limited to TLS 1.2, we have to use esp-mbedtls, which requires certificate verification. 

No worries--this is a great opportunity to introduce an alternative approach as well.

In contrast to standard environments, where the operating system provides a built-in set of trusted certificate chains (and the browser or HTTP client handles verification automatically), our embedded environment lacks this support. Therefore, we must manually obtain the TLS certificate from the website and store it in a file, so our program can use it for verification.

You can run this command and store the file in the `src` folder.

```sh
openssl s_client -showcerts -connect openweathermap.org:443 </dev/null 2>/dev/null | \
                   awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/' > ca_cert.pem
```

Alternatively, you can download it from my project repository [here](https://github.com/ImplFerris/esp32-epaper-weather/blob/main/src/ca_cert.pem), though this isn't recommended as the file may be outdated.

## Weather API

Let's create a wrapper struct to handle API requests. While a simple function could work, it would require passing the URL and Wi-Fi stack every time. By using a struct, we can instantiate it once and call the access_website method whenever we need the latest weather data.

```rust
pub struct WeatherApi {
    wifi: Stack<'static>,
    url: String<120>,
}
```

In the new function, we'll construct the API URL by appending the API key and store it in the field.

NOTE: If you observed, I have used "London" as the city name. You can replace it with your city name along with the country code. Refer to the doc for more details and adjust accordingly.

```rust
pub fn new(wifi: Stack<'static>) -> Self {
        let mut url = String::new();
        url.push_str(
            "https://api.openweathermap.org/data/2.5/weather?q=London&units=metric&appid=",
        )
        .unwrap();
        url.push_str(API_KEY).unwrap();
        Self { wifi, url }
    }
```

## Sending API Request

Next, we'll instantiate the DNS socket and TCP client. When configuring TLS, we'll specify the TLS 1.2 protocol and set up the certificate using the CA chain. The certificate content will be loaded from the "ca_cert.pem" file we created earlier.

With the basic setup in place, we'll pass the TCP client, DNS socket, and TLS configuration to the HttpClient, and use it to send a request to the API URL. Once we receive the response, we'll deserialize the JSON payload into a WeatherData struct using serde_json_core.

```rust
pub async fn access_website(&self, tls_reference: TlsReference<'_>) -> WeatherData {
        let dns = DnsSocket::new(self.wifi);
        let tcp_state = TcpClientState::<1, 4096, 4096>::new();
        let tcp = TcpClient::new(self.wifi, &tcp_state);

        let tls_config = TlsConfig::new(
            reqwless::TlsVersion::Tls1_2,
            reqwless::Certificates {
                ca_chain: reqwless::X509::pem(
                    concat!(include_str!("./ca_cert.pem"), "\0").as_bytes(),
                )
                .ok(),
                ..Default::default()
            },
            tls_reference,
        );

        let mut client = HttpClient::new_with_tls(&tcp, &dns, tls_config);
        let mut buffer = [0u8; 4096];
        let mut http_req = client
            .request(reqwless::request::Method::GET, &self.url)
            .await
            .unwrap();
        let response = http_req.send(&mut buffer).await.unwrap();

        info!("Got response");
        let res = response.body().read_to_end().await.unwrap();

        let (data, _): (WeatherData, _) = serde_json_core::de::from_slice(res).unwrap();
        data
    }
```

## The rest of the struct

One thing to note here: the API gives the weather condition as a number (it also provides it as a string, but the number is easier to work with). We'll map those numbers to our ConditionCode enum. We've also added an icon function to that enum, which tells us which icon to show for each specific weather condition.


```rust
#[derive(Debug, Deserialize_repr)]
#[repr(u16)]
pub enum ConditionCode {
    // Group 2xx: Thunderstorm
    ThunderstormWithLightRain = 200,
    ThunderstormWithRain = 201,
    ThunderstormWithHeavyRain = 202,
    LightThunderstorm = 210,
    Thunderstorm = 211,
    HeavyThunderstorm = 212,
    RaggedThunderstorm = 221,
    ThunderstormWithLightDrizzle = 230,
    ThunderstormWithDrizzle = 231,
    ThunderstormWithHeavyDrizzle = 232,

    // Group 3xx: Drizzle
    LightIntensityDrizzle = 300,
    Drizzle = 301,
    HeavyIntensityDrizzle = 302,
    LightIntensityDrizzleRain = 310,
    DrizzleRain = 311,
    HeavyIntensityDrizzleRain = 312,
    ShowerRainAndDrizzle = 313,
    HeavyShowerRainAndDrizzle = 314,
    ShowerDrizzle = 321,

    // Group 5xx: Rain
    LightRain = 500,
    ModerateRain = 501,
    HeavyIntensityRain = 502,
    VeryHeavyRain = 503,
    ExtremeRain = 504,
    FreezingRain = 511,
    LightIntensityShowerRain = 520,
    ShowerRain = 521,
    HeavyIntensityShowerRain = 522,
    RaggedShowerRain = 531,

    // Group 6xx: Snow
    LightSnow = 600,
    Snow = 601,
    HeavySnow = 602,
    Sleet = 611,
    LightShowerSleet = 612,
    ShowerSleet = 613,
    LightRainAndSnow = 615,
    RainAndSnow = 616,
    LightShowerSnow = 620,
    ShowerSnow = 621,
    HeavyShowerSnow = 622,

    // Group 7xx: Atmosphere
    Mist = 701,
    Smoke = 711,
    Haze = 721,
    SandDustWhirls = 731,
    Fog = 741,
    Sand = 751,
    Dust = 761,
    VolcanicAsh = 762,
    Squalls = 771,
    Tornado = 781,

    // Group 800: Clear
    ClearSky = 800,

    // Group 80x: Clouds
    FewClouds = 801,
    ScatteredClouds = 802,
    BrokenClouds = 803,
    OvercastClouds = 804,
}

impl ConditionCode {
    pub fn icon(&self) -> &'static str {
        match self {
            // Thunderstorm
            ConditionCode::ThunderstormWithLightRain
            | ConditionCode::ThunderstormWithRain
            | ConditionCode::ThunderstormWithHeavyRain
            | ConditionCode::LightThunderstorm
            | ConditionCode::Thunderstorm
            | ConditionCode::HeavyThunderstorm
            | ConditionCode::RaggedThunderstorm
            | ConditionCode::ThunderstormWithLightDrizzle
            | ConditionCode::ThunderstormWithDrizzle
            | ConditionCode::ThunderstormWithHeavyDrizzle => "storm.bmp",

            // Drizzle
            ConditionCode::LightIntensityDrizzle
            | ConditionCode::Drizzle
            | ConditionCode::HeavyIntensityDrizzle
            | ConditionCode::LightIntensityDrizzleRain
            | ConditionCode::DrizzleRain
            | ConditionCode::HeavyIntensityDrizzleRain
            | ConditionCode::ShowerRainAndDrizzle
            | ConditionCode::HeavyShowerRainAndDrizzle
            | ConditionCode::ShowerDrizzle => "rainy.bmp",

            // Rain
            ConditionCode::LightRain
            | ConditionCode::ModerateRain
            | ConditionCode::HeavyIntensityRain
            | ConditionCode::VeryHeavyRain
            | ConditionCode::ExtremeRain
            | ConditionCode::LightIntensityShowerRain
            | ConditionCode::ShowerRain
            | ConditionCode::HeavyIntensityShowerRain
            | ConditionCode::RaggedShowerRain => "rainy_heavy.bmp",
            ConditionCode::FreezingRain => "weather_mix.bmp",

            // Snow
            ConditionCode::LightSnow
            | ConditionCode::Snow
            | ConditionCode::HeavySnow
            | ConditionCode::Sleet
            | ConditionCode::LightShowerSleet
            | ConditionCode::ShowerSleet
            | ConditionCode::LightRainAndSnow
            | ConditionCode::RainAndSnow
            | ConditionCode::LightShowerSnow
            | ConditionCode::ShowerSnow
            | ConditionCode::HeavyShowerSnow => "snowing.bmp",

            // Atmosphere
            ConditionCode::Mist
            | ConditionCode::Smoke
            | ConditionCode::Haze
            | ConditionCode::SandDustWhirls
            | ConditionCode::Fog
            | ConditionCode::Sand
            | ConditionCode::Dust
            | ConditionCode::VolcanicAsh
            | ConditionCode::Squalls => "foggy.bmp",
            ConditionCode::Tornado => "cyclone.bmp",

            // Clear
            ConditionCode::ClearSky => "sunny.bmp",

            // Clouds
            ConditionCode::FewClouds
            | ConditionCode::ScatteredClouds
            | ConditionCode::BrokenClouds
            | ConditionCode::OvercastClouds => "partly_cloudy_day.bmp",
        }
    }
}

#[derive(Debug, Deserialize)]
pub struct Weather {
    pub id: ConditionCode,
}

#[derive(Debug, Deserialize)]
pub struct Main {
    pub temp: f64,
    pub feels_like: f64,
    pub temp_min: f64,
    pub temp_max: f64,
    pub pressure: i32,
    pub humidity: i32,
    pub sea_level: Option<i32>,
    pub grnd_level: Option<i32>,
}

#[derive(Debug, Deserialize)]
pub struct Wind {
    pub speed: f64,
    pub deg: f64,
    pub gust: Option<f64>,
}

```
