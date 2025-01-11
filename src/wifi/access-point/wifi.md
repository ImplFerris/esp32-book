# Setup ESP32 Wi-Fi (in wifi.rs file)
 
To create our own Wi-Fi network with ESP32, we need to set a static IP address using CIDR notation(eg: 192.168.2.1/24) and specify the gateway IP(e.g: 192.168.2.1). We also need to give a Wi-Fi name (SSID), which can be anything you like, as long as it's within 32 characters. While the password is optional for the network, we will set it up with a password. The SSID and password will be loaded from environment variables, just like we did in Station mode.

```rust
// Unlike Station mode, You can give any IP range(private) that you like
// IP Address/Subnet mask eg: STATIC_IP=192.168.13.37/24
const STATIC_IP: &str = "192.168.13.37/24";
// Gateway IP eg: GATEWAY_IP="192.168.13.37"
const GATEWAY_IP: &str = "192.168.13.37";

const PASSWORD: &str = env!("PASSWORD");
const SSID: &str = env!("SSID");
```

## Start Wi-Fi Function

This is the function we called from the main function.  This is almost similar to what we have used in previous exercises, except we will use WifiApDevice for the Access Point mode. In this function, we will do the following:

1. We create the Wi-Fi interface and controller in Access Point mode .

2. We parse the static IP and gateway address, which are loaded from environment variables. Then, we create a network configuration instance using these values.

3. Using the network configuration, we create the network stack and the runner instance. 

4. We spawn the connection task, which monitors the Wi-Fi network. If the connection stops, it restarts. We also spawn the network task, which handles all network communications.

5. We wait for the link to be up. Once it's ready, we print the IP address, which should be the static IP address we configured.

6. Finally, we will return the stack instance which will be used by the web task

```rust
pub async fn start_wifi(
    wifi_init: &'static EspWifiController<'static>,
    wifi: esp_hal::peripherals::WIFI,
    mut rng: Rng,
    spawner: &Spawner,
) -> anyhow::Result<&'static Stack<'static>> {

    // Create Wifi interface and controller
    let (wifi_interface, controller) =
        esp_wifi::wifi::new_with_mode(wifi_init, wifi, WifiApDevice).unwrap();
    let net_seed = rng.random() as u64 | ((rng.random() as u64) << 32);

    // Parse STATIC_IP
    let ip_addr =
        Ipv4Cidr::from_str(STATIC_IP).map_err(|_| anyhow!("Invalid STATIC_IP: {}", STATIC_IP))?;

    // Parse GATEWAY_IP
    let gateway = Ipv4Addr::from_str(GATEWAY_IP)
        .map_err(|_| anyhow!("Invalid GATEWAY_IP: {}", GATEWAY_IP))?;

    // Create Network config with IP details
    let net_config = embassy_net::Config::ipv4_static(StaticConfigV4 {
        address: ip_addr,
        gateway: Some(gateway),
        dns_servers: Default::default(),
    });

    // alternate approach
    // let net_config = embassy_net::Config::ipv4_static(StaticConfigV4 {
    //     address: Ipv4Cidr::new(Ipv4Address::new(192, 168, 2, 1), 24),
    //     gateway: Some(Ipv4Address::from_bytes(&[192, 168, 2, 1])),
    //     dns_servers: Default::default(),
    // });

    // Create Network stack and the runner
    let (stack, runner) = mk_static!(
        (
            Stack<'static>,
            Runner<'static, WifiDevice<'static, WifiApDevice>>
        ),
        embassy_net::new(
            wifi_interface,
            net_config,
            mk_static!(StackResources<3>, StackResources::<3>::new()),
            net_seed
        )
    );

    // spawn the tasks
    spawner.spawn(connection_task(controller)).ok();
    spawner.spawn(net_task(runner)).ok();

    // wait for link to be up and print the IP
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

    Ok(stack)
}
```

### Network Task

```rust
#[embassy_executor::task]
async fn net_task(runner: &'static mut Runner<'static, WifiDevice<'static, WifiApDevice>>) -> ! {
    runner.run().await
}
```

### Connection Task

The main goal of this function is to ensure the Wi-Fi network is running. If it is not running, it will be restarted. The function checks the Wi-Fi state in a loop. If the state is `ApStarted`, it waits until the Wi-Fi gets stopped (i.e., the `ApStop` event occurs). Once that happens, it moves to the second part of the loop.

If the Wi-Fi is not started, we will configure the Wi-Fi with the SSID, an optional password, and WPA2 Personal authentication mode. Then, we will start the Wi-Fi network.

**Note:** 

If you want to run the Wi-Fi without a password, you can comment out the `password` and `auth_method` fields in the `AccessPointConfiguration`. This will make the Wi-Fi network passwordless, will use the default configuration.


```rust
#[embassy_executor::task]
async fn connection_task(mut controller: WifiController<'static>) {
    println!("start connection task");
    println!("Device capabilities: {:?}", controller.capabilities());

    loop {
        if esp_wifi::wifi::wifi_state() == WifiState::ApStarted {
            // wait until we're no longer connected
            controller.wait_for_event(WifiEvent::ApStop).await;
            Timer::after(Duration::from_millis(5000)).await
        }

        if !matches!(controller.is_started(), Ok(true)) {
            let wifi_config = Configuration::AccessPoint(AccessPointConfiguration {
                ssid: SSID.try_into().unwrap(), // Use whatever Wi-Fi name you want
                password: PASSWORD.try_into().unwrap(), // Set your password
                auth_method: esp_wifi::wifi::AuthMethod::WPA2Personal,
                ..Default::default()
            });
            controller.set_configuration(&wifi_config).unwrap();
            println!("Starting wifi");
            controller.start_async().await.unwrap();
            println!("Wifi started!");
        }
    }
}
```
