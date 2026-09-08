{{#title Access Websites Asynchronously on ESP32 Using Embassy}}

# Final: Access the Website

We have covered how to set up the Wi-Fi connection with Embassy. Since the previous chapter became quite lengthy, we didn't dive into the details of the access_website function. In this chapter, we will focus on how to send an HTTP request and receive a response from a website using the reqwless crate, and then we finally will run the code.


### Sending HTTP Request

We begin by initializing the DNS socket and the TCP client.

**TLS Config:**

Next, we set up the TLS configuration using the tls_seed, which is the random number we generated earlier and passed into the function. We configure TLS to skip SSL certificate verification by setting TlsVerify::None. While this is not recommended for production environments, we use this insecure setting for simplicity.  In production, you would need to configure TLS with certificate verification and provide the certificate chain so that it can use for verification. 

The remaining part is pretty straightforward. We initialize the HTTP client with the TCP client, DNS socket, and TLS config. Then, we send a GET request to the jsonplaceholder website, which returns JSON content. We read the content and then print it to the console.

```rust
async fn access_website(stack: Stack<'_>, tls_seed: u64) {
    let mut rx_buffer = [0; 4096];
    let mut tx_buffer = [0; 4096];
    let dns = DnsSocket::new(stack);
    let tcp_state = TcpClientState::<1, 4096, 4096>::new();
    let tcp = TcpClient::new(stack, &tcp_state);

    let tls = TlsConfig::new(
        tls_seed,
        &mut rx_buffer,
        &mut tx_buffer,
        reqwless::client::TlsVerify::None,
    );

    let mut client = HttpClient::new_with_tls(&tcp, &dns, tls);
    let mut buffer = [0u8; 4096];
    let mut http_req = client
        .request(
            reqwless::request::Method::GET,
            "https://jsonplaceholder.typicode.com/posts/1",
        )
        .await
        .unwrap();
    let response = http_req.send(&mut buffer).await.unwrap();

    info!("Got response");
    let res = response.body().read_to_end().await.unwrap();

    let content = core::str::from_utf8(res).unwrap();
    println!("{}", content);
}
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `wifi-async-http` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/wifi-async-http
```

### How to run?

Normally, we would simply run `cargo run --release`, but this time we also need to pass the environment variables for the Wi-Fi connection.

```sh
SSID=YOUR_WIFI_NAME PASSWORD=YOUR_WIFI_PASSWORD  cargo run --release
```


## The Full code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]

use defmt::info;
use embassy_executor::Spawner;
use embassy_net::{
    DhcpConfig, Runner, Stack, StackResources,
    dns::DnsSocket,
    tcp::client::{TcpClient, TcpClientState},
};
use embassy_time::{Duration, Timer};
use esp_hal::clock::CpuClock;
use esp_hal::rng::Rng;
use esp_hal::timer::timg::TimerGroup;
use esp_println::{self as _, println};
use esp_radio::wifi::{
    AuthenticationMethodConfig, Config, ControllerConfig, Interface, WifiController,
    sta::StationConfig,
};
use reqwless::client::{HttpClient, TlsConfig};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

// If you are okay with using a nightly compiler, you can use the macro provided by the static_cell crate: https://docs.rs/static_cell/2.1.0/static_cell/macro.make_static.html

macro_rules! mk_static {
    ($t:ty,$val:expr) => {{
        static STATIC_CELL: static_cell::StaticCell<$t> = static_cell::StaticCell::new();
        #[deny(unused_attributes)]
        let x = STATIC_CELL.uninit().write(($val));
        x
    }};
}

const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");

#[esp_hal::main]
async fn main(spawner: Spawner) -> ! {
    // generator version: 1.0.0

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    esp_alloc::heap_allocator!(#[unsafe(link_section = ".dram2_uninit")] size: 98767);

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0, peripherals.FROM_CPU_INTR0);

    info!("Embassy initialized!");

    let rng = Rng::new();
    let station_config = Config::Station(
        StationConfig::default()
            .with_ssid(SSID.try_into().unwrap())
            .with_authentication(AuthenticationMethodConfig::Wpa2Personal(
                PASSWORD.try_into().unwrap(),
            )),
    );

    let wifi_interface = Interface::station();
    let wifi_controller = WifiController::new(
        peripherals.WIFI,
        ControllerConfig::default().with_initial_config(station_config),
    )
    .expect("Failed to initialize Wi-Fi controller");

    let net_seed = rng.random() as u64 | ((rng.random() as u64) << 32);
    let tls_seed = rng.random() as u64 | ((rng.random() as u64) << 32);

    let dhcp_config = DhcpConfig::default();
    let config = embassy_net::Config::dhcpv4(dhcp_config);

    // Init network stack
    let (stack, runner) = embassy_net::new(
        wifi_interface,
        config,
        mk_static!(StackResources<3>, StackResources::<3>::new()),
        net_seed,
    );

    spawner.spawn(connection(wifi_controller).unwrap());
    spawner.spawn(net_task(runner).unwrap());

    wait_for_connection(stack).await;

    access_website(stack, tls_seed).await;

    loop {}
}

async fn wait_for_connection(stack: Stack<'_>) {
    stack.wait_config_up().await;
    if let Some(config) = stack.config_v4() {
        println!("Got IP: {}", config.address);
    }
}

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

#[embassy_executor::task]
async fn net_task(mut runner: Runner<'static, Interface>) {
    runner.run().await
}

async fn access_website(stack: Stack<'_>, tls_seed: u64) {
    let mut rx_buffer = [0; 4096];
    let mut tx_buffer = [0; 4096];
    let dns = DnsSocket::new(stack);
    let tcp_state = TcpClientState::<1, 4096, 4096>::new();
    let tcp = TcpClient::new(stack, &tcp_state);

    let tls = TlsConfig::new(
        tls_seed,
        &mut rx_buffer,
        &mut tx_buffer,
        reqwless::client::TlsVerify::None,
    );

    let mut client = HttpClient::new_with_tls(&tcp, &dns, tls);
    let mut buffer = [0u8; 4096];
    let mut http_req = client
        .request(
            reqwless::request::Method::GET,
            "https://jsonplaceholder.typicode.com/posts/1",
        )
        .await
        .unwrap();
    let response = http_req.send(&mut buffer).await.unwrap();

    info!("Got response");
    let res = response.body().read_to_end().await.unwrap();

    let content = core::str::from_utf8(res).unwrap();
    println!("{}", content);
}

```
