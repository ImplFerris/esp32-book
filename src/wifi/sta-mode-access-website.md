# How to Connect ESP32 to Wi-Fi and Access Websites

In this exercise, we will configure the ESP32 in STA mode to connect to your Wi-Fi. Then, we will make a request to a web server from the ESP32 and print the response in the system console. This code is based on the esp-hal examples.

### Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 wifi-website
```

This will open a screen asking you to select options. 

- Select the option "Enables Wi-Fi via the esp-wifi crate. Requires alloc".  It automatically selects the espa-alloc crate option also

Just save it by pressing "s" in the keyboard.

### Update dependencies

Add the following crate to the Cargo.toml file
```toml
blocking-network-stack = { git = "https://github.com/bjoernQ/blocking-network-stack.git", rev = "1c581661d78e0cf0f17b936297179b993fb149d7" }
```

blocking-network-stack is a simple crate that provides Non-async Networking primitives for TCP/UDP communication.

## esp-alloc crate

"A simple no_std heap allocator for RISC-V and Xtensa processors from Espressif. Supports all currently available ESP32 devices."

When we choose the Wi-Fi option, it also includes the esp_alloc crate. This crate enables heap allocation for esp-wifi to manage memory dynamically. esp-wifi uses an implementation of malloc() and free(), and it uses Rust's Global allocator. The esp-alloc crate provides the global allocator for ESP SoCs.

We initialize the heap with a size of 72 KiB (72 * 1024) using the esp_alloc::heap_allocator! macro.

```rust
esp_alloc::heap_allocator!(72 * 1024);
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

let esp_wifi_ctrlr = esp_wifi::init(timg0.timer0, rng.clone(), peripherals.RADIO_CLK).unwrap();
```

Next, we will call the create_network_interface function with the initialized esp_wifi_ctrlr, the Wi-Fi peripheral instance, and the Wi-Fi mode we want to use, which is STA (Station).

```rust
let mut wifi = peripherals.WIFI;
let (iface, device, mut controller) =
    create_network_interface(&esp_wifi_ctrlr, &mut wifi, WifiStaDevice).unwrap();
```

**Wi-Fi Operation Mode**:


Next, we configure the Wi-Fi operation mode using the Configuration enum. Since we want the ESP32 to act as a Wi-Fi client, we use the Client variant. We need to provide the SSID (Service Set Identifier, which is the name of your Wi-Fi network) and the Wi-Fi password.

```rust
let wifi_config = Configuration::Client(ClientConfiguration {
    ssid: SSID.try_into().unwrap(),
    password: PASSWORD.try_into().unwrap(),
    ..Default::default()
});

let res = controller.set_configuration(&wifi_config);
```

Start the wifi controller
```rust
controller.start().unwrap();
```

## Wi-Fi Scanning and Connecting to the Access Point

Next, we perform a blocking Wi-Fi network scan with the default options. This means the program will pause and wait until the scan is complete. Once the scan is finished, it will return a list of available Wi-Fi access points. We then loop through the results and print each access point's information to the console.

```rust
let res: Result<(heapless::Vec<AccessPointInfo, 10>, usize), WifiError> = controller.scan_n();

if let Ok((res, _count)) = res {
    for ap in res {
        println!("{:?}", ap);
    }
}

```

Then, we print the supported capabilities of the controller, which will show only "Client" since that's what we configured.

```rust
println!("{:?}", controller.capabilities());
```

Finally, we call the connect method on the Wi-Fi controller. This will connect the ESP32 to the Wi-Fi network that we specified earlier.

```rust
println!("wifi_connect {:?}", controller.connect());
```

We will continuously check in a loop if the Wi-Fi is connected. Once it connects successfully, we will proceed further.

```rust
loop {
        match controller.is_connected() {
            Ok(true) => break,
            Ok(false) => {}
            Err(err) => {
                println!("{:?}", err);
                loop {}
            }
        }
    }
```

At this point, the ESP32 should be successfully connected to the Wi-Fi network. Next, we will set up the necessary code to send an HTTP request to a website. Once the request is sent, the ESP32 will receive the HTML response from the website, and we will print the HTML content to the console.


## HTTP Client

To send a web request to a website, we need an HTTP client. For this, we will use the [smoltcp crate](https://crates.io/crates/smoltcp), to send a raw HTTP request over a TCP socket. 

"smoltcp is a standalone, event-driven TCP/IP stack that is designed for bare-metal, real-time systems.  smoltcp does not need heap allocation at all, is extensively documented, and compiles on stable Rust 1.80 and later."


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
    data: b"implrust",
}]);
socket_set.add(dhcp_socket);
```

### Initializing the Network Stack

Let's initialize the Stack from the smoltcp crate using the network interface and device (which we obtained from the Wi-Fi network interface we created earlier), the sockets, a closure to get time, and a random number.


```rust
let now = || time::now().duration_since_epoch().to_millis();
let stack = Stack::new(iface, device, socket_set, now, rng.random());
```


### Obtain IP address
We need to obtain an IP address, so we first wait for the DHCP process to complete. We will continuously loop until the interface is up and we have successfully obtained an IP address.

```rust
 // wait for getting an ip address
    println!("Wait to get an ip address");
    loop {
        stack.work();

        if stack.is_iface_up() {
            println!("got ip {:?}", stack.get_ip_info());
            break;
        }
    }
```

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
socket.work();

socket
    .open(IpAddress::Ipv4(Ipv4Address::new(142, 250, 185, 115)), 80)
    .unwrap();

socket
    .write(b"GET / HTTP/1.0\r\nHost: www.mobile-j.de\r\n\r\n")
    .unwrap();
socket.flush().unwrap();
```

### Read the response
Next, we read the response from the server with a 20-second timeout. 
```rust
let deadline = time::now() + Duration::secs(20);
let mut buffer = [0u8; 512];

while let Ok(len) = socket.read(&mut buffer) {
    let to_print = unsafe { core::str::from_utf8_unchecked(&buffer[..len]) };
    print!("{}", to_print);

    if time::now() > deadline {
        println!("Timeout");
        break;
    }
}
println!();
```

### Close the socket
After the HTTP request is complete, we close the socket and wait for 5 seconds.

```rust
socket.disconnect();

let deadline = time::now() + Duration::secs(5);
while time::now() < deadline {
    socket.work();
}
```


## Clone the existing project
You can also clone (or refer) project I created and navigate to the `wifi-website` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/wifi-website
```

### How to run?

Normally, we would simply run `cargo run --release`, but this time we also need to pass the environment variables for the Wi-Fi connection.

```sh
SSID=YOUR_WIFI_NAME PASSWORD=YOUR_WIFI_PASSWORD  cargo run --release
```
