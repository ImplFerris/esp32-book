# Connecting ESP32 to Wi-Fi with Embassy support

So, we have discussed the dependencies we needed for this exercise and the features that needed to be enabled. Next, we'll focus on the coding. First, we will look at the code to connect Wi-Fi using Embassy.

## Helper Macro for StaticCell

In an embedded environment, the StaticCell crate is useful when you need to initialize a variable at runtime but require it to have a static lifetime.  We will define a macro to create globally accessible static variables.  This macro takes two arguments: the type of the variable and the value to initialize it with. The uninit function provides a mutable reference to the uninitialized memory, and we write the value into it.

```rust
// If you are okay with using a nightly compiler, you can use the macro provided by the static_cell crate: https://docs.rs/static_cell/2.1.0/static_cell/macro.make_static.html

macro_rules! mk_static {
    ($t:ty,$val:expr) => {{
        static STATIC_CELL: static_cell::StaticCell<$t> = static_cell::StaticCell::new();
        #[deny(unused_attributes)]
        let x = STATIC_CELL.uninit().write(($val));
        x
    }};
}
```

## Initializing the Wi-Fi Controller

Let's initialize Embassy with the usual setup:

```rust
let timg0 = TimerGroup::new(peripherals.TIMG0);
esp_rtos::start(timg0.timer0);
```

Load the Wi-Fi credentials from environment variables:
```rust
const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");
```

Next, we initialize the Radio controller with a static lifetime. We use the `mk_static!` macro to promote the controller to `'static` lifetime because the Wi-Fi network stack will run as an asynchronous task that processes network events continuously throughout the program's execution. This requires the `radio_init` variable to remain valid for the entire duration of the program.

```rust
// let radio_init = esp_radio::init().expect("Failed to initialize Wi-Fi/BLE controller");
let radio_init = &*mk_static!(
    esp_radio::Controller<'static>,
    esp_radio::init().expect("Failed to initialize Wi-Fi/BLE controller")
);
```

Next, we create the Wi-Fi controller along with its associated network interfaces, then extract the STA interface.

```rust
let (wifi_controller, interfaces) =
        esp_radio::wifi::new(&radio_init, peripherals.WIFI, Default::default())
            .expect("Failed to initialize Wi-Fi controller");
let wifi_interface = interfaces.sta;
```

### Initialize the network stack

We need a random number for both the TLS configuration and network stack initialization, and both require a u64. However, since rng generates only u32 values, we generate two random numbers and place one in the most significant bits (MSB) and the other in the least significant bits (LSB) using bitwise operation:

```rust
let rng = Rng::new();
let net_seed = rng.random() as u64 | ((rng.random() as u64) << 32);
let tls_seed = rng.random() as u64 | ((rng.random() as u64) << 32);
```

Let's initialize the network stack from the embassy_net crate using the network interface obtained from the Wi-Fi controller, a random number as the seed, the DHCP configuration, and stack resources with a size of 3.

```rust
 let dhcp_config = DhcpConfig::default();
// dhcp_config.hostname = Some(String::from_str("implRust").unwrap());

let config = embassy_net::Config::dhcpv4(dhcp_config);
// Init network stack
let (stack, runner) = embassy_net::new(
    wifi_interface,
    config,
    mk_static!(StackResources<3>, StackResources::<3>::new()),
    net_seed,
);
```

Next, we will start two background tasks: the connection_task will maintain the Wi-Fi connection, while the net_task will run the network stack and handle network events.

```rust
spawner.spawn(connection(controller)).ok();
spawner.spawn(net_task(runner)).ok();
```

We'll shortly discuss what happens in these two tasks and check these function definitions. But first, let's complete the flow.

## Wait for Wi-Fi connection

We will wait for the Wi-Fi link to be up, then obtain the IP address.

```rust
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


## Wi-Fi connection tasks

The connection_task function manages the Wi-Fi connection by continuously checking the status, configuring the Wi-Fi controller, and attempting to reconnect if the connection is lost or not started.

1. First, we check the Wi-Fi state. If it is in StaConnected, we wait there until it gets disconnected. If it gets disconnected, we move to the other steps in the loop.
2. We check if the Wi-Fi controller is started. If not, we initialize the Wi-Fi client configuration with the SSID (Wi-Fi name) and password, and start it.
3. Finally, we attempt to connect to the Wi-Fi.

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

### Run the network stack 

```rust
#[embassy_executor::task]
async fn net_task(mut runner: Runner<'static, WifiDevice<'static>>) {
    runner.run().await
}
```
