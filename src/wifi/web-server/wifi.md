{{#title ESP32 Wi-Fi Connection Module with Embassy and Rust}}

# Wi-Fi Module


## Wi-Fi connection setup

The Wi-Fi setup code is the same as explained in the ["Connecting Wi-Fi with Embassy"](../embassy/connecting-wifi.md) chapter on the Access Website. To avoid repetition, I won't explain it again here. Please refer to that chapter if you haven't already.

```rust
use crate::mk_static;
use embassy_executor::Spawner;
use embassy_net::{DhcpConfig, Runner, Stack, StackResources};
use embassy_time::{Duration, Timer};
use esp_hal::rng::Rng;
use esp_println::println;
use esp_radio::wifi::{
    AuthenticationMethodConfig, Config, ControllerConfig, Interface, WifiController,
    sta::StationConfig,
};

const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");
```

## Start Wi-Fi

The `start_wifi` function is responsible for setting up and starting the Wi-Fi connection for the ESP32. Here's a step-by-step explanation:

1. We will create the Wi-Fi interface and controller in STA (Station) mode so the ESP32 can connect to an existing Wi-Fi network.

2. Normally, when a device connects to a Wi-Fi network, the router assigns it an IP address automatically using  DHCP Server. In this exercise, we will configure the ESP32 to request DHCP for IP.  
   To achieve this, we will create a `net_config` instance configured for DHCP. Then, we will use this configuration along with the Wi-Fi interface to initialize the network stack and runner instances.

3. We will spawn two tasks:
   - The **connection task** to monitor the Wi-Fi connection and reconnect, if it is disconnected.
   - The **network task** to manage all network communications.

4.  We will wait until the Wi-Fi link is up. Once the connection is ready, we will print the IP address assigned to our ESP32.

5.  Finally, we will return the network stack instance. This will be used later by the web task to handle web server operations.

```rust

pub async fn start_wifi(
    wifi: esp_hal::peripherals::WIFI<'static>,
    rng: Rng,
    spawner: &Spawner,
) -> Stack<'static> {
    let station_config = Config::Station(
        StationConfig::default()
            .with_ssid(SSID.try_into().unwrap())
            .with_authentication(AuthenticationMethodConfig::Wpa2Personal(
                PASSWORD.try_into().unwrap(),
            )),
    );
    let wifi_interface = Interface::station();
    let wifi_controller = WifiController::new(
        wifi,
        ControllerConfig::default().with_initial_config(station_config),
    )
    .expect("Failed to initialize Wi-Fi controller");
    let net_seed = rng.random() as u64 | ((rng.random() as u64) << 32);

    let dhcp_config = DhcpConfig::default();
    let net_config = embassy_net::Config::dhcpv4(dhcp_config);

    // Init network stack
    let (stack, runner) = embassy_net::new(
        wifi_interface,
        net_config,
        mk_static!(StackResources<3>, StackResources::<3>::new()),
        net_seed,
    );

    spawner.spawn(connection(wifi_controller).unwrap());
    spawner.spawn(net_task(runner).unwrap());

    wait_for_connection(stack).await;

    stack
}

async fn wait_for_connection(stack: Stack<'_>) {
    stack.wait_config_up().await;
    if let Some(config) = stack.config_v4() {
        println!("Got IP: {}", config.address);
    }
}
```


### Wi-Fi and Network Tasks

The connection task maintains the Wi-Fi connection, while the network task continuously runs the `embassy-net` stack.

```rust
#[embassy_executor::task]
async fn connection(mut controller: WifiController<'static>) {
    println!("start connection task");
    loop {
        println!("About to connect...");

        match controller.connect_async().await {
            Ok(info) => {
                println!("Wifi connected to {:?}", info);
                let info = controller.wait_for_disconnect_async().await.ok();
                println!("Disconnected: {:?}", info);
            }
            Err(e) => {
                println!("Failed to connect to wifi: {:?}", e);
            }
        }

        Timer::after(Duration::from_millis(5000)).await;
    }
}
```

```rust
#[embassy_executor::task]
async fn net_task(mut runner: Runner<'static, Interface>) {
    runner.run().await
}
```
