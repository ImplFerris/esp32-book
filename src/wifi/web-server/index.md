# Writing Rust Code to Run a Website on ESP32

In earlier exercises, we accessed websites and printed the response in the console. In this exercise, we'll do the reverse: We will run a web server on the ESP32. This server will be accessible on the local Wi-Fi network. To make it accessible from the internet, additional setup is required. First, we'll focus on accessing the site within the local Wi-Fi network. We will still be working in STA mode only (i.e connecting to existing Wi-Fi).

## What We will Be Doing
We'll set up a simple web server to serve a single index.html page. For this example, let's assume the ESP32 has been assigned the IP address "192.168.0.101" (it gets displayed it in the console, when we are running). Once the server is running, you can access the page by navigating to 'http://192.168.0.101/' in your computer's browser. You can either use your own HTML page or the index.html page I created for this exercise, which you can find [here](https://github.com/ImplFerris/esp32-projects/blob/main/webserver-html/src/bin/index.html).

<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">DHCP vs Static IP </b>
        <p>You can set a static IP address instead of letting the DHCP server assign it. This makes the IP address consistent but adds some extra steps. To keep things simple, we won't do it in this exercise. We’ll show you how to set up a static IP in a later exercise.</p>
    </div>
</div>

No more waiting, let's start right away. 

## Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 webserver-html
```

This will open a screen asking you to select options. 

- Select the option "Enables Wi-Fi via the esp-wifi crate. Requires alloc".  It automatically selects the espa-alloc crate option also
- Select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.


## Update dependencies

### picoserve crate
[picoserve](https://docs.rs/picoserve/latest/picoserve/) is a crate that provides an asynchronous HTTP server designed for bare-metal environments, heavily inspired by Axum. As you might have guessed from the name, it was first created with "Raspberry Pi Pico W" and embassy in mind. But it works fine with other embedded runtimes and hardware, including the ESP32. This crate makes our lives much easier. Without it, we would have to build the web server core from scratch, a time-consuming task that would be beyond the scope of this book.

```toml
picoserve = { version = "0.13.3", features = ["embassy"] }
```

### Task arena size update
We will update the embassy-executor with the task-arena-size-65536 feature. For more details, refer to the Task Arena Size documentation [here](https://docs.embassy.dev/embassy-executor/git/cortex-m/index.html#task-arena).

```toml
embassy-executor = { version = "0.6.3", features = ["task-arena-size-65536"] }
```

### Update the embassy-net
To make some functions compatible with the latest picoserve crate, I needed to update embassy-net to version 0.5.0.

```toml
embassy-net = { version = "0.5.0", features = [
    "tcp",
    "udp",
    "dhcpv4",
    "medium-ethernet",
] }
```


## Wi-Fi connection setup

The Wi-Fi setup code is the same as explained in the ["Connecting Wi-Fi with Embassy"](../embassy/connecting-wifi.md) chapter on the Access Website. To avoid repetition, I won't explain it again here. Please refer to that chapter if you haven't already.

```rust
const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");

macro_rules! mk_static {
    ($t:ty,$val:expr) => {{
        static STATIC_CELL: static_cell::StaticCell<$t> = static_cell::StaticCell::new();
        #[deny(unused_attributes)]
        let x = STATIC_CELL.uninit().write(($val));
        x
    }};
}
```

In the main function, we have included boilerplate code to set up the global heap allocator and initialize Embassy. After that, we have created a Wi-Fi controller and run the network stack along with Wi-Fi connection monitoring tasks in the background.  Finally, we print the IP address assigned to our ESP32 by DHCP.

```rust
let peripherals = esp_hal::init({
    let mut config = esp_hal::Config::default();
    config.cpu_clock = CpuClock::max();
    config
});

esp_alloc::heap_allocator!(72 * 1024);

esp_println::logger::init_logger_from_env();

let timer0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG1);
esp_hal_embassy::init(timer0.timer0);

let mut rng = Rng::new(peripherals.RNG);
let net_seed = rng.random() as u64 | ((rng.random() as u64) << 32);

info!("Embassy initialized!");

let timg0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG0);

let wifi_init = &*mk_static!(
    EspWifiController<'static>,
    esp_wifi::init(timg0.timer0, rng.clone(), peripherals.RADIO_CLK).unwrap()
);

let wifi = peripherals.WIFI;

let (wifi_interface, controller) =
    esp_wifi::wifi::new_with_mode(&wifi_init, wifi, WifiStaDevice).unwrap();

let dhcp_config = DhcpConfig::default();

let net_config = embassy_net::Config::dhcpv4(dhcp_config);

//  This part different from previous:
let (stack, runner) = mk_static!(
    (
        Stack<'static>,
        Runner<'static, WifiDevice<'static, WifiStaDevice>>
    ),
    embassy_net::new(
        wifi_interface,
        net_config,
        mk_static!(StackResources<3>, StackResources::<3>::new()),
        net_seed
    )
);

spawner.spawn(connection_task(controller)).ok();
spawner.spawn(net_task(runner)).ok();

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

```

The code is almost the same as what we did earlier, with a small change. In the version of embassy-net we are using, we have to initialize the network stack by calling `embassy_net::new`, which returns a tuple containing the stack and runner instances.

### Wi-Fi and Network Tasks

There is no major change in the logic of these two tasks. The only difference is that we are now passing the runner instance to the net_task, unlike before.

```rust
#[embassy_executor::task]
async fn connection_task(mut controller: WifiController<'static>) {
    println!("start connection task");
    println!("Device capabilities: {:?}", controller.capabilities());
    loop {
        match esp_wifi::wifi::wifi_state() {
            WifiState::StaConnected => {
                // wait until we're no longer connected
                controller.wait_for_event(WifiEvent::StaDisconnected).await;
                Timer::after(Duration::from_millis(5000)).await
            }
            _ => {}
        }
        if !matches!(controller.is_started(), Ok(true)) {
            let client_config = Configuration::Client(ClientConfiguration {
                ssid: SSID.try_into().unwrap(),
                password: PASSWORD.try_into().unwrap(),
                ..Default::default()
            });
            controller.set_configuration(&client_config).unwrap();
            println!("Starting wifi");
            controller.start_async().await.unwrap();
            println!("Wifi started!");
        }
        println!("About to connect...");

        match controller.connect_async().await {
            Ok(_) => println!("Wifi connected!"),
            Err(e) => {
                println!("Failed to connect to wifi: {e:?}");
                Timer::after(Duration::from_millis(5000)).await
            }
        }
    }
}

#[embassy_executor::task]
async fn net_task(runner: &'static mut Runner<'static, WifiDevice<'static, WifiStaDevice>>) -> ! {
    runner.run().await
}
```
