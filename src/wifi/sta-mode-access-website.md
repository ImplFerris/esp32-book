{{#title How to Connect ESP32 to Wi-Fi in Rust}}

# How to Connect ESP32 to Wi-Fi

In this exercise, we will configure the ESP32 in station (STA) mode and connect it to an existing Wi-Fi network. We will use Embassy for asynchronous tasks and `embassy-net` for the network stack.

## Generate a project using esp-generate

Create a new project with `esp-generate`:

```sh
esp-generate --chip esp32 wifi-webfetch
```

Enable the following options:

- Enable unstable HAL features.
- Enable allocations via the `esp-alloc` crate.
- Enable Wi-Fi via the `esp-radio` crate.
- Add Embassy framework support.

You can also enable `defmt` logging under the flashing, logging, and debugging options. Press `s` to save and generate the project.

## Wi-Fi credentials

We need to provide the name and password of the Wi-Fi network. Instead of hardcoding them, we will read them from environment variables at compile time:

```rust
const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");
```

## Initialize Embassy

Create a timer and use one of the software interrupts to start the Embassy runtime:

```rust
let timg0 = TimerGroup::new(peripherals.TIMG0);
esp_rtos::start(timg0.timer0, peripherals.FROM_CPU_INTR0);
```

## Configure station mode

The station configuration contains the SSID and authentication settings for the network we want to join. In this example, we use WPA2 Personal authentication.

```rust
use esp_radio::wifi::{
    AuthenticationMethodConfig,
    Config,
    ControllerConfig,
    Interface,
    WifiController,
    sta::StationConfig,
};

let station_config = Config::Station(
    StationConfig::default()
        .with_ssid(SSID.try_into().unwrap())
        .with_authentication(AuthenticationMethodConfig::Wpa2Personal(
            PASSWORD.try_into().unwrap(),
        )),
);
```

`try_into()` converts the SSID and password into the fixed-capacity string types used by `esp-radio`. It returns an error if either value is too long, so `unwrap()` is acceptable here when the credentials are known at compile time.

## Create the Wi-Fi interface and controller

`Interface::station()` creates the network device that will be passed to `embassy-net`. The `WifiController` manages the radio and receives the station configuration.

```rust
let wifi_interface = Interface::station();
let wifi_controller = WifiController::new(
    peripherals.WIFI,
    ControllerConfig::default().with_initial_config(station_config),
)
.expect("Failed to initialize Wi-Fi controller");
```

The controller starts Wi-Fi using the supplied configuration. We can then move it into an Embassy task that handles the connection.

## Connect and reconnect

The connection task attempts to join the configured network. After a successful connection, it waits until the ESP32 is disconnected. The loop then waits for five seconds before trying again.

```rust
#[embassy_executor::task]
async fn connection(mut controller: WifiController<'static>) {
    println!("start connection task");

    loop {
        println!("About to connect...");

        match controller.connect_async().await {
            Ok(info) => {
                println!("Wi-Fi connected to {:?}", info);
                let info = controller.wait_for_disconnect_async().await.ok();
                println!("Disconnected: {:?}", info);
            }
            Err(error) => {
                println!("Failed to connect to Wi-Fi: {:?}", error);
            }
        }

        Timer::after(Duration::from_secs(5)).await;
    }
}
```

## Initialize the network stack

Connecting to an access point establishes the Wi-Fi link. To use TCP, UDP, DNS, and DHCP, we also create an `embassy-net` stack with the station interface.

```rust
let net_config = embassy_net::Config::dhcpv4(Default::default());
let rng = Rng::new();
let seed = (rng.random() as u64) << 32 | rng.random() as u64;

let (stack, runner) = embassy_net::new(
    wifi_interface,
    net_config,
    mk_static!(StackResources<3>, StackResources::<3>::new()),
    seed,
);

spawner.spawn(connection(wifi_controller).unwrap());
spawner.spawn(net_task(runner).unwrap());
```

The network runner must continue running in the background:

```rust
#[embassy_executor::task]
async fn net_task(mut runner: Runner<'static, Interface>) {
    runner.run().await
}
```

Finally, wait for DHCP to provide a network configuration and print the assigned address:

```rust
stack.wait_config_up().await;

if let Some(config) = stack.config_v4() {
    println!("Got IP: {}", config.address);
}
```

Continue to [Access Website with Embassy](./embassy/async-access-website.md) to add DNS and make an HTTP request.

## Run the program

Pass the Wi-Fi credentials when building and flashing the application:

```sh
SSID=YOUR_WIFI_NAME PASSWORD=YOUR_WIFI_PASSWORD cargo run --release
```
