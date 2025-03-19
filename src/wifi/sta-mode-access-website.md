# How to Connect ESP32 to Wi-Fi and Access Websites

In this exercise, we will configure the ESP32 in STA mode to connect to your Wi-Fi. Then, we will make a request to a web server from the ESP32 and print the response in the system console. This code is based on the esp-hal examples.

### Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 wifi-webfetch
```

This will open a screen asking you to select options. In order to Enable Wi-Fi, we will first need to enable "unstable" and "alloc" features. If you noticed, until you select these two options, you wont be able to enable Wi-Fi option. So select one by one

- First, select the option "Enable unstable HAL features."
- Select the option "Enable allocations via the esp-alloc crate."
- Now, you can enable "Enable Wi-Fi via esp-wifi crate."


Enable the logging feature also

- So, scroll to "Flashing, logging and debugging (espflash)" and hit Enter.
- Then, Select "Use defmt to print messages".

Just save it by pressing "s" in the keyboard.

### Update dependencies

Add the following crate to the Cargo.toml file
```toml
blocking-network-stack = { git = "https://github.com/bjoernQ/blocking-network-stack.git", rev = "b3ecefc222d8806edd221f266999ca339c52d34e", default-features = false, features = [
  "dhcpv4",
  "tcp",
] }
```

blocking-network-stack is a simple crate that provides Non-async Networking primitives for TCP/UDP communication.

## esp-alloc crate

"A simple no_std heap allocator for RISC-V and Xtensa processors from Espressif. Supports all currently available ESP32 devices."

When we choose the Wi-Fi option, it also includes the esp_alloc crate. This crate enables heap allocation for esp-wifi to manage memory dynamically. esp-wifi uses an implementation of malloc() and free(), and it uses Rust's Global allocator. The esp-alloc crate provides the global allocator for ESP SoCs.


We will create a helper function that initializes the ESP HAL and returns the peripherals instance. We will also set up a 72 KiB heap using the esp_alloc::heap_allocator! macro.

```rust
fn init_hardware() -> Peripherals {
    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);
    esp_alloc::heap_allocator!(size: 72 * 1024);
    peripherals
}
```

## Initializing the Wi-Fi Controller

Load the Wi-Fi credentials from environment variables:
```rust
const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");
```

First, we initialize the TimerGroup and the Random Number Generator (RNG) required for setting up the Wi-Fi controller. The RNG is also used to initialize the non-async TCP/IP network stack, so we will use clone so that we can re-use it. 

```rust
let timg0 = TimerGroup::new(peripherals.TIMG0);
let mut rng = Rng::new(peripherals.RNG);

```

First, we initialize the WiFi controller using a hardware timer, RNG, and clock peripheral. Next, we create a WiFi driver instance (controller to manage connections and interfaces for network modes). Finally, we configure the device to use station (STA) mode , allowing it to connect to WiFi networks as a client.

```rust
let esp_wifi_ctrl = esp_wifi::init(timg0.timer0, rng.clone(), peripherals.RADIO_CLK).unwrap();
let (mut controller, interfaces) =
    esp_wifi::wifi::new(&esp_wifi_ctrl, peripherals.WIFI).unwrap();
let mut device = interfaces.sta;
```


### SocketSet Initialization
We will create a SocketSet with storage for up to 3 sockets to manage multiple sockets, such as DHCP and TCP, within the stack.

```rust
let mut socket_set_entries: [SocketStorage; 3] = Default::default();
let mut socket_set = SocketSet::new(&mut socket_set_entries[..]);
```

### DHCP Socket

We will create a DHCP socket to request an IP address from a DHCP server. We can set the outgoing options, including the hostname using DHCP Client Option 12 (you can read more about DHCP options [here](https://efficientip.com/glossary/dhcp-option/)). We will also add the socket to the SocketSet we created earlier. 

```rust
let mut dhcp_socket = smoltcp::socket::dhcpv4::Socket::new();

// we can set a hostname here (or add other DHCP options)
dhcp_socket.set_outgoing_options(&[DhcpOption {
    kind: 12,
    data: b"implRust",
}]);
socket_set.add(dhcp_socket);
```

### Initializing the Network Stack

Let's initialize the Stack from the smoltcp crate using the network interface and device (which we obtained from the Wi-Fi network interface we created earlier), the sockets, a closure to get time, and a random number.


```rust
let now = || time::Instant::now().duration_since_epoch().as_millis();
let mut stack = Stack::new(
    create_interface(&mut device),
    device,
    socket_set,
    now,
    rng.random(),
);
```

Boilerplate functions to set up a network interface for the smoltcp TCP/IP stack using the Wi-Fi device (taken from the official esp-hal examples)
```rust
pub fn create_interface(device: &mut esp_wifi::wifi::WifiDevice) -> smoltcp::iface::Interface {
    // users could create multiple instances but since they only have one WifiDevice
    // they probably can't do anything bad with that
    smoltcp::iface::Interface::new(
        smoltcp::iface::Config::new(smoltcp::wire::HardwareAddress::Ethernet(
            smoltcp::wire::EthernetAddress::from_bytes(&device.mac_address()),
        )),
        device,
        timestamp(),
    )
}

// some smoltcp boilerplate
fn timestamp() -> smoltcp::time::Instant {
    smoltcp::time::Instant::from_micros(
        esp_hal::time::Instant::now()
            .duration_since_epoch()
            .as_micros() as i64,
    )
}
```


**Wi-Fi Operation Mode**:

Next, we configure the Wi-Fi operation mode using the Configuration enum. Since we want the ESP32 to act as a Wi-Fi client, we use the Client variant. We need to provide the SSID (Service Set Identifier, which is the name of your Wi-Fi network) and the Wi-Fi password.

```rust
wifi::Configuration::Client(wifi::ClientConfiguration {
    ssid: SSID.try_into().unwrap(),
    password: PASSWORD.try_into().unwrap(),
    ..Default::default()
});

let res = controller.set_configuration(&client_config);
info!("wifi_set_configuration returned {:?}", res);

// Start the wifi controller
controller.start().unwrap();
```

## Wi-Fi Scanning and Connecting to the Access Point

Next, we perform a blocking Wi-Fi network scan with the default options. This means the program will pause and wait until the scan is complete. Once the scan is finished, it will return a list of available Wi-Fi access points. We then loop through the results and print each access point's information to the console.

```rust
fn scan_wifi(controller: &mut WifiController<'_>) {
    info!("Start Wifi Scan");
    let res: Result<(heapless::Vec<_, 10>, usize), _> = controller.scan_n();
    if let Ok((res, _count)) = res {
        for ap in res {
            info!("{:?}", ap);
        }
    }
}
```

## Connect Wi-Fi
Then, we print the supported capabilities of the controller, which will show only "Client" since that's what we configured.

Next, we call the connect method on the Wi-Fi controller. This will connect the ESP32 to the Wi-Fi network that we specified earlier.

We will continuously check in a loop if the Wi-Fi is connected. Once it connects successfully, we will proceed further.

```rust
fn connect_wifi(
    controller: &mut WifiController<'_>,
    stack: &mut Stack<'_, esp_wifi::wifi::WifiDevice<'_>>,
) {
    println!("{:?}", controller.capabilities());
    info!("wifi_connect {:?}", controller.connect());

    info!("Wait to get connected");
    loop {
        match controller.is_connected() {
            Ok(true) => break,
            Ok(false) => {}
            Err(err) => panic!("{:?}", err),
        }
    }
    info!("Connected: {:?}", controller.is_connected());

    info!("Wait for IP address");
    loop {
        stack.work();
        if stack.is_iface_up() {
            println!("IP acquired: {:?}", stack.get_ip_info());
            break;
        }
    }
}
```

### Obtain IP address
We need to obtain an IP address, so we first wait for the DHCP process to complete. We will continuously loop until the interface is up and we have successfully obtained an IP address.

```rust
fn obtain_ip(stack: &mut Stack<'_, esp_wifi::wifi::WifiDevice<'_>>) {
    info!("Wait for IP address");
    loop {
        stack.work();
        if stack.is_iface_up() {
            println!("IP acquired: {:?}", stack.get_ip_info());
            break;
        }
    }
}
```

At this point, the ESP32 should be successfully connected to the Wi-Fi network. Next, we will set up the necessary code to send an HTTP request to a website. Once the request is sent, the ESP32 will receive the HTML response from the website, and we will print the HTML content to the console.


## HTTP Client

To send a web request to a website, we need an HTTP client. For this, we will use the [smoltcp crate](https://crates.io/crates/smoltcp), to send a raw HTTP request over a TCP socket. 

"smoltcp is a standalone, event-driven TCP/IP stack that is designed for bare-metal, real-time systems.  smoltcp does not need heap allocation at all, is extensively documented, and compiles on stable Rust 1.80 and later."


### TCP Socket

We will initialize the TCP socket with read and write buffers to handle incoming and outgoing data for the socket.

```rust
let mut rx_buffer = [0u8; 1536];
let mut tx_buffer = [0u8; 1536];
let mut socket = stack.get_socket(&mut rx_buffer, &mut tx_buffer);
```

### Send Get Request
Let's send an HTTP request with GET method to the website "www.mobile-j.de". This website is used in the esp-hal examples, which returns "Hello fellow Rustaceans!" with ferris ascii art. We'll use the IP address of the website to open a connection on port 80, then send the HTTP request.

```rust
info!("Making HTTP request");
socket.work();

let remote_addr = IpAddress::v4(142, 250, 185, 115);
socket.open(remote_addr, 80).unwrap();
socket
    .write(b"GET / HTTP/1.0\r\nHost: www.mobile-j.de\r\n\r\n")
    .unwrap();
socket.flush().unwrap();
```

### Read the response
Next, we read the response from the server with a 20-second timeout. 
```rust
let deadline = Instant::now() + Duration::from_secs(20);
let mut buffer = [0u8; 512];
while let Ok(len) = socket.read(&mut buffer) {
    // let text = unsafe { core::str::from_utf8_unchecked(&buffer[..len]) };
    let Ok(text) = core::str::from_utf8(&buffer[..len]) else {
        panic!("Invalid UTF-8 sequence encountered");
    };

    info!("{}", text);

    if Instant::now() > deadline {
        info!("Timeout");
        break;
    }
}
```

### Close the socket
After the HTTP request is complete, we close the socket and wait for 5 seconds.

```rust
socket.disconnect();
let deadline = Instant::now() + Duration::from_secs(5);
while Instant::now() < deadline {
    socket.work();
}
```


## Clone the existing project
You can also clone (or refer) project I created and navigate to the `wifi-webfetch` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/wifi-webfetch
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

use blocking_network_stack::Stack;
use defmt::info;
use embedded_io::{Read, Write};
use esp_hal::clock::CpuClock;
use esp_hal::delay::Delay;
use esp_hal::peripherals::Peripherals;
use esp_hal::rng::Rng;
use esp_hal::time::{Duration, Instant};
use esp_hal::timer::timg::TimerGroup;
use esp_hal::{main, time};
use esp_println as _;
use esp_println::println;
use esp_wifi::wifi::{self, WifiController};
use smoltcp::iface::{SocketSet, SocketStorage};
use smoltcp::wire::{DhcpOption, IpAddress};

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

extern crate alloc;

const SSID: &str = env!("SSID");
const PASSWORD: &str = env!("PASSWORD");

#[main]
fn main() -> ! {
    // generator version: 0.3.1

    // https://github.com/esp-rs/esp-hal/blob/esp-hal-v1.0.0-beta.0/examples/src/bin/wifi_dhcp.rs

    let peripherals = init_hardware();

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let mut rng = Rng::new(peripherals.RNG);

    let esp_wifi_ctrl = esp_wifi::init(timg0.timer0, rng.clone(), peripherals.RADIO_CLK).unwrap();
    let (mut controller, interfaces) =
        esp_wifi::wifi::new(&esp_wifi_ctrl, peripherals.WIFI).unwrap();
    let mut device = interfaces.sta;

    // let mut stack = setup_network_stack(device, &mut rng);
    let mut socket_set_entries: [SocketStorage; 3] = Default::default();
    let mut socket_set = SocketSet::new(&mut socket_set_entries[..]);
    let mut dhcp_socket = smoltcp::socket::dhcpv4::Socket::new();

    // we can set a hostname here (or add other DHCP options)
    dhcp_socket.set_outgoing_options(&[DhcpOption {
        kind: 12,
        data: b"implRust",
    }]);
    socket_set.add(dhcp_socket);

    let now = || time::Instant::now().duration_since_epoch().as_millis();
    let mut stack = Stack::new(
        create_interface(&mut device),
        device,
        socket_set,
        now,
        rng.random(),
    );

    configure_wifi(&mut controller);
    scan_wifi(&mut controller);
    connect_wifi(&mut controller);
    obtain_ip(&mut stack);

    let mut rx_buffer = [0u8; 1536];
    let mut tx_buffer = [0u8; 1536];
    let mut socket = stack.get_socket(&mut rx_buffer, &mut tx_buffer);

    http_loop(&mut socket)
}

pub fn create_interface(device: &mut esp_wifi::wifi::WifiDevice) -> smoltcp::iface::Interface {
    // users could create multiple instances but since they only have one WifiDevice
    // they probably can't do anything bad with that
    smoltcp::iface::Interface::new(
        smoltcp::iface::Config::new(smoltcp::wire::HardwareAddress::Ethernet(
            smoltcp::wire::EthernetAddress::from_bytes(&device.mac_address()),
        )),
        device,
        timestamp(),
    )
}

// some smoltcp boilerplate
fn timestamp() -> smoltcp::time::Instant {
    smoltcp::time::Instant::from_micros(
        esp_hal::time::Instant::now()
            .duration_since_epoch()
            .as_micros() as i64,
    )
}

fn init_hardware() -> Peripherals {
    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);
    esp_alloc::heap_allocator!(size: 72 * 1024);
    peripherals
}

fn configure_wifi(controller: &mut WifiController<'_>) {
    let client_config = wifi::Configuration::Client(wifi::ClientConfiguration {
        ssid: SSID.try_into().unwrap(),
        password: PASSWORD.try_into().unwrap(),
        ..Default::default()
    });

    let res = controller.set_configuration(&client_config);
    info!("wifi_set_configuration returned {:?}", res);

    controller.start().unwrap();
    info!("is wifi started: {:?}", controller.is_started());
}

fn scan_wifi(controller: &mut WifiController<'_>) {
    info!("Start Wifi Scan");
    let res: Result<(heapless::Vec<_, 10>, usize), _> = controller.scan_n();
    if let Ok((res, _count)) = res {
        for ap in res {
            info!("{:?}", ap);
        }
    }
}

fn connect_wifi(controller: &mut WifiController<'_>) {
    println!("{:?}", controller.capabilities());
    info!("wifi_connect {:?}", controller.connect());

    info!("Wait to get connected");
    loop {
        match controller.is_connected() {
            Ok(true) => break,
            Ok(false) => {}
            Err(err) => panic!("{:?}", err),
        }
    }
    info!("Connected: {:?}", controller.is_connected());
}

fn obtain_ip(stack: &mut Stack<'_, esp_wifi::wifi::WifiDevice<'_>>) {
    info!("Wait for IP address");
    loop {
        stack.work();
        if stack.is_iface_up() {
            println!("IP acquired: {:?}", stack.get_ip_info());
            break;
        }
    }
}

fn http_loop(
    socket: &mut blocking_network_stack::Socket<'_, '_, esp_wifi::wifi::WifiDevice<'_>>,
) -> ! {
    info!("Starting HTTP client loop");
    let delay = Delay::new();
    loop {
        info!("Making HTTP request");
        socket.work();

        let remote_addr = IpAddress::v4(142, 250, 185, 115);
        socket.open(remote_addr, 80).unwrap();
        socket
            .write(b"GET / HTTP/1.0\r\nHost: www.mobile-j.de\r\n\r\n")
            .unwrap();
        socket.flush().unwrap();

        let deadline = Instant::now() + Duration::from_secs(20);
        let mut buffer = [0u8; 512];
        while let Ok(len) = socket.read(&mut buffer) {
            // let text = unsafe { core::str::from_utf8_unchecked(&buffer[..len]) };
            let Ok(text) = core::str::from_utf8(&buffer[..len]) else {
                panic!("Invalid UTF-8 sequence encountered");
            };

            info!("{}", text);

            if Instant::now() > deadline {
                info!("Timeout");
                break;
            }
        }

        socket.disconnect();
        let deadline = Instant::now() + Duration::from_secs(5);
        while Instant::now() < deadline {
            socket.work();
        }

        delay.delay_millis(1000);
    }
}
```
