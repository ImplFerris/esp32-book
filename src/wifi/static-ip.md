
# Static IP Address

In previous exercises, we have been relying on the DHCP server to assign an IP address to the ESP32. However, this can be unreliable as the IP address may change over time. In many cases, we want to assign a static IP address to ensure the ESP32 always has a fixed, predictable address. This makes it easier to access the device consistently without having to check or update its IP address every time it reconnects to the network.

Now, we will remove the DHCP-related code and modify the net_config variable to initialize with a static IP address instead.

First, let's define the constants that will load the static IP and gateway IP from environment variables. The IP address should be in CIDR format, which includes both the IP address and the subnet mask. We need to specify the IP address followed by a slash and the subnet mask. For example, if you want to assign the IP address 192.168.0.50 to your ESP32, you should write it as 192.168.0.50/24.


<div class="alert-box alert-box-info">
    <span class="icon"><i class="fa fa-info"></i></span>
    <div class="alert-content">
        <b class="alert-title">Finding IP</b>
        <p>You can't assign just any IP address. You need to find the IP range your Wi-Fi router is using. To do this, you can type `ip a` in the terminal and look for the IP address next to your Wi-Fi interface (typically starting with `wl`). For example, if your system's IP address is 192.168.0.103, you can assign an IP address starting from 192.168.0.2</p>
    </div>
</div>


```rust
// IP Address/Subnet mask eg: STATIC_IP=192.168.0.50/24
const STATIC_IP: &str = env!("STATIC_IP");
const GATEWAY_IP: &str = env!("GATEWAY_IP");
```

You will also need to configure the gateway, although it's not required for this exercise since we won't be sending requests to the internet. However, it's good practice to configure it for future exercises. 

The gateway address is often the first address in your Wi-Fi IP range. For instance, if your IP addresses range from 192.168.0.1 to 192.168.0.255, the gateway is likely to be 192.168.0.1.  You can also use the command `ip route | grep default` in linux to find your gateway address.

```rust
//find the `let net_config` part and replace
let Ok(ip_addr) = Ipv4Cidr::from_str(STATIC_IP) else {
    println!("Invalid STATIC_IP");
    loop {}
};

let Ok(gateway) = Ipv4Addr::from_str(GATEWAY_IP) else {
    println!("Invalid GATEWAY_IP");
    loop {}
};

let net_config = embassy_net::Config::ipv4_static(StaticConfigV4 {
    address: ip_addr,
    gateway: Some(gateway),
    dns_servers: Vec::new(),
});
// You dont need to change anything in `embassy_net::new` call.
```


## Project Base

I have reorganized the project by splitting it into modules like wifi and web to keep the main file clean. We will use this project as the base for the upcoming exercise, so I recommend taking a look at how it is organized.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/webserver-base
```

**Project Structure**:

```
├── build.rs
├── Cargo.toml
├── rust-toolchain.toml
├── src
│   ├── bin
│   │   └── async_main.rs
│   ├── index.html
│   ├── lib.rs
│   ├── web.rs
│   └── wifi.rs
```

### How to run?

Normally, we would simply run `cargo run --release`, but this time we also need to pass the environment variables for the Wi-Fi connection.

```sh
SSID=YOUR_WIFI_NAME PASSWORD=YOUR_WIFI_PASSWORD STATIC_IP=ASSIGN_ESP32_IP/24 GATEWAY_IP=WIFI_GATEWAY_IP  cargo run --release
```
