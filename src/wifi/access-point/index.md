# Access Point - Create Wi-Fi network on ESP32

So far, we have been using an existing Wi-Fi network. However, you can create your own Wi-Fi network with the ESP32 (just don't expect it to provide internet 😉). In this exercise, we will configure the ESP32 as an access point and run the web server.


## Project base

We will once again copy the webserver-base project and work on top of that. 

I recommend you to read these section before you proceed furhter; This will avoid unnecessary repetition of code and explanations.
- [Creating Web Server](../web-server/index.md) 
 

```sh
git clone https://github.com/ImplFerris/esp32-projects
cp -r esp32-projects/webserver-base ~/YOUR_PROJECT_FOLDER/wifi-led
```

## Setup ESP32 Wi-Fi (in wifi.rs file)
 
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

## Update the connection task

The main goal of this function is to ensure the Wi-Fi network is running. If it is not running, it will be restarted. The function checks the Wi-Fi state in a loop. If the state is `WifiApState::Started`, it waits until the Wi-Fi gets stopped (i.e., the `ApStop` event occurs). If that happens, it moves to the second part of the loop.

If the Wi-Fi is not started, we will configure the Wi-Fi with the SSID, an optional password, and WPA2 Personal authentication mode. Then, we will start the Wi-Fi network.

**Note:** 

If you want to run the Wi-Fi without a password, you can comment out the `password` and `auth_method` builder in the `AccessPointConfig`. This will make the Wi-Fi network passwordless, will use the default configuration.


```rust
#[embassy_executor::task]
async fn connection(mut controller: WifiController<'static>) {
    println!("start connection task");
    println!("Device capabilities: {:?}", controller.capabilities());
    loop {
        match esp_radio::wifi::ap_state() {
            WifiApState::Started => {
                // wait until we're no longer connected
                controller.wait_for_event(WifiEvent::ApStop).await;
                Timer::after(Duration::from_millis(5000)).await
            }
            _ => {}
        }

        if !matches!(controller.is_started(), Ok(true)) {
            let client_config = ModeConfig::AccessPoint(
                AccessPointConfig::default()
                    .with_ssid(SSID.into())
                    .with_password(PASSWORD.into())
                    .with_auth_method(esp_radio::wifi::AuthMethod::Wpa2Personal),
            );
            controller.set_config(&client_config).unwrap();
            println!("Starting wifi");
            controller.start_async().await.unwrap();
            println!("Wifi started!");
        }
    }
}
```

## Update the start_wifi Function

We need to change the interface to use Access Point mode instead of Station mode.

```rust
let wifi_interface = interfaces.ap;
```
