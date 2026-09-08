{{#title Async Wi-Fi HTTP Requests on ESP32 with Embassy and Rust}}

# Connecting ESP32 to Wi-Fi and Accessing Websites Asynchronously with Embassy

In this exercise, we will use STA mode to connect to a Wi-Fi network and access a website asynchronously. We will use Embassy for asynchronous tasks and the `reqwless` crate as our HTTP client.


## Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 wifi-async-http
```

This will open a screen asking you to select options. In order to Enable Wi-Fi, we will first need to enable "unstable" and "alloc" features. If you noticed, until you select these two options, you wont be able to enable Wi-Fi option. So select one by one

- First, select the option "Enable unstable HAL features."
- Select the option "Enable allocations via the esp-alloc crate."
- Now, you can enable "Enable Wi-Fi via esp-radio crate."
- Select the option "Adds embassy framework support".

Enable the logging feature also

- So, scroll to "Flashing, logging and debugging (espflash)" and hit Enter.
- Then, Select "Use defmt to print messages".

Just save it by pressing "s" in the keyboard.

## Update dependencies

### Embassy net

embassy-net, built on the smoltcp crate, provides an async network stack for embedded systems. It offers a higher-level API and supports IPv4, IPv6, Ethernet, bare-IP mediums, TCP, UDP, DNS, DHCPv4, and multicast. 

This crate automatically gets added when you select the Wi-Fi and Embassy option while generating the project with esp-generate. However, we need one additional feature: "dns". This is necessary to perform HTTP requests because the website name (e.g., google.com) needs to be resolved into an IP address. Remember, in the previous exercise, we manually provided the IP address of the website.

```toml
# Updated embassy-net in Cargo.toml
embassy-net = { version = "0.9.0", features = [
  "defmt",
  "dhcpv4",
  "medium-ethernet",
  "tcp",
  "udp",
  #addition:
  "dns",
] }
```

`embassy-net` uses `smoltcp` internally. The Wi-Fi `Interface` can be passed directly to `embassy-net`, which will use it as the network device.

### Reqwless
The reqwless crate is an HTTP client for embedded systems, working with any transport that implements traits from the embedded-io crate.

```toml
reqwless = { version = "0.13.0", default-features = false, features = [
    "embedded-tls",
] }
```
