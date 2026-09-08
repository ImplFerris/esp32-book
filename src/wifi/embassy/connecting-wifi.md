{{#title Learn Async Wi-Fi Networking with ESP32 and Embassy}}

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
esp_rtos::start(timg0.timer0, peripherals.FROM_CPU_INTR0);
```

Load the Wi-Fi credentials from environment variables:
```rust
const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");
```

Next, create the station configuration. We provide the SSID and configure WPA2 Personal authentication using the Wi-Fi password.

```rust
let station_config = Config::Station(
    StationConfig::default()
        .with_ssid(SSID.try_into().unwrap())
        .with_authentication(AuthenticationMethodConfig::Wpa2Personal(
            PASSWORD.try_into().unwrap(),
        )),
);
```

Now, create the station interface and Wi-Fi controller. The interface will be used by the network stack, while the controller configures and manages the Wi-Fi radio. We pass the station configuration as the initial configuration so that the controller starts in station mode.

```rust
let wifi_interface = Interface::station();
let wifi_controller = WifiController::new(
    peripherals.WIFI,
    ControllerConfig::default().with_initial_config(station_config),
)
.expect("Failed to initialize Wi-Fi controller");
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
spawner.spawn(connection(wifi_controller).unwrap());
spawner.spawn(net_task(runner).unwrap());
```

We'll shortly discuss what happens in these two tasks and check these function definitions. But first, let's complete the flow.

## Wait for network configuration

Wait until `embassy-net` has a usable configuration, then print the address assigned by DHCP.

```rust
async fn wait_for_connection(stack: Stack<'_>) {
    stack.wait_config_up().await;
    if let Some(config) = stack.config_v4() {
        println!("Got IP: {}", config.address);
    }
}
```


## Wi-Fi connection tasks

The `connection` task connects to the configured access point and reconnects after a disconnection.

After `connect_async()` succeeds, the task waits for a disconnection. If the connection is lost, the loop waits briefly and then attempts to connect again.

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

### Run the network stack 

```rust
#[embassy_executor::task]
async fn net_task(mut runner: Runner<'static, Interface>) {
    runner.run().await
}
```
