{{#title ESP32 Access Point Mode and Web Server in Embedded Rust}}

# Access Point - Create Wi-Fi network on ESP32

So far, we have been using an existing Wi-Fi network. However, you can create your own Wi-Fi network with the ESP32 (just don't expect it to provide internet). In this exercise, we will configure the ESP32 as an access point and run the web server.


## Project base

We will once again copy the `webserver-base` project and work on top of that.

I recommend reading this section before you proceed further. This will avoid unnecessary repetition of code and explanations.

- [Creating Web Server](../web-server/index.md) 
 

```sh
git clone https://github.com/ImplFerris/esp32-projects
cp -r esp32-projects/webserver-base ~/YOUR_PROJECT_FOLDER/wifi-led
```

## Setup ESP32 Wi-Fi (in wifi.rs file)
 
To create our own Wi-Fi network with the ESP32, we need to set a static IP address using CIDR notation (for example, `192.168.2.1/24`) and specify the gateway IP (for example, `192.168.2.1`). We also need to give the network a Wi-Fi name (SSID), which can be anything you like as long as it is no longer than 32 characters. The password is optional, but we will configure one for this exercise. The SSID and password will be loaded from environment variables, just as they were in station mode.

```rust
// Unlike station mode, you can use any suitable private IP range.
// IP address/subnet mask, for example: STATIC_IP=192.168.13.37/24
const STATIC_IP: &str = "192.168.13.37/24";
// Gateway IP, for example: GATEWAY_IP="192.168.13.37"
const GATEWAY_IP: &str = "192.168.13.37";

const PASSWORD: &str = env!("PASSWORD");
const SSID: &str = env!("SSID");
```

## Configure the access point

First, create an access-point configuration with the network name, password, and authentication method. We will pass this configuration to the Wi-Fi controller when it is initialized.

In this example, we will use WPA2 Personal authentication. `AuthenticationMethodConfig::Wpa2Personal` receives the password for the network.

**Note:** 

If you want to create a network without a password, leave out `with_authentication`. The default access-point configuration creates an open network.


```rust
use esp_radio::wifi::{
    AuthenticationMethodConfig, Config, ControllerConfig, Interface, WifiController,
    ap::AccessPointConfig,
};

let access_point_config = Config::AccessPoint(
    AccessPointConfig::default()
        .with_ssid(SSID.try_into().unwrap())
        .with_authentication(AuthenticationMethodConfig::Wpa2Personal(
            PASSWORD.try_into().unwrap(),
        )),
);

let wifi_interface = Interface::access_point();
let wifi_controller = WifiController::new(
    wifi,
    ControllerConfig::default().with_initial_config(access_point_config),
)
.expect("Failed to initialize Wi-Fi controller");
```

## Monitor connected stations

The connection task waits for another device to connect to or disconnect from the ESP32 access point. We match the event and print its details to the console:

```rust
#[embassy_executor::task]
async fn connection(controller: WifiController<'static>) {
    loop {
        match controller.wait_for_access_point_connected_event_async().await {
            Ok(esp_radio::wifi::ap::EventInfo::Connected(info)) => {
                println!("Station connected: {:?}", info);
            }
            Ok(esp_radio::wifi::ap::EventInfo::Disconnected(info)) => {
                println!("Station disconnected: {:?}", info);
            }
            Err(error) => println!("Wi-Fi event error: {:?}", error),
        }
    }
}
```
