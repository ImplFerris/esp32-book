# Wi-Fi Module


## Wi-Fi connection setup

The Wi-Fi setup code is the same as explained in the ["Connecting Wi-Fi with Embassy"](../embassy/connecting-wifi.md) chapter on the Access Website. To avoid repetition, I won't explain it again here. Please refer to that chapter if you haven't already.

```rust
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
    radio_init: &'static esp_radio::Controller<'static>,
    wifi: esp_hal::peripherals::WIFI<'static>,
    rng: Rng,
    spawner: &Spawner,
) -> Stack<'static> {
    let (wifi_controller, interfaces) = esp_radio::wifi::new(radio_init, wifi, Default::default())
        .expect("Failed to initialize Wi-Fi controller");

    let wifi_interface = interfaces.sta;
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

    spawner.spawn(connection(wifi_controller)).ok();
    spawner.spawn(net_task(runner)).ok();

    wait_for_connection(stack).await;

    stack
}

async fn wait_for_connection(stack: Stack<'_>) {
    println!("Waiting for link to be up");
    loop {
        if stack.is_link_up() {
            break;
        }
        Timer::after(Duration::from_millis(500)).await;
    }

    println!("Waiting to get IP address...");
    loop {
        if let Some(config) = stack.config_v4() {
            println!("Got IP: {}", config.address);
            break;
        }
        Timer::after(Duration::from_millis(500)).await;
    }
}
```


### Wi-Fi and Network Tasks

There is no major change in the logic of these two tasks. The only difference is that we are now passing the runner instance to the net_task, unlike before.

```rust
#[embassy_executor::task]
async fn connection(mut controller: WifiController<'static>) {
    println!("start connection task");
    println!("Device capabilities: {:?}", controller.capabilities());
    loop {
        match esp_radio::wifi::sta_state() {
            WifiStaState::Connected => {
                // wait until we're no longer connected
                controller.wait_for_event(WifiEvent::StaDisconnected).await;
                Timer::after(Duration::from_millis(5000)).await
            }
            _ => {}
        }
        if !matches!(controller.is_started(), Ok(true)) {
            let client_config = ModeConfig::Client(
                ClientConfig::default()
                    .with_ssid(SSID.into())
                    .with_password(PASSWORD.into()),
            );
            controller.set_config(&client_config).unwrap();
            println!("Starting wifi");
            controller.start_async().await.unwrap();
            println!("Wifi started!");

            println!("Scan");
            let scan_config = ScanConfig::default().with_max(10);
            let result = controller
                .scan_with_config_async(scan_config)
                .await
                .unwrap();
            for ap in result {
                println!("{:?}", ap);
            }
        }
        println!("About to connect...");

        match controller.connect_async().await {
            Ok(_) => println!("Wifi connected!"),
            Err(e) => {
                println!("Failed to connect to wifi: {:?}", e);
                Timer::after(Duration::from_millis(5000)).await
            }
        }
    }
}
```

```rust
#[embassy_executor::task]
async fn net_task(mut runner: Runner<'static, WifiDevice<'static>>) {
    runner.run().await
}
```
