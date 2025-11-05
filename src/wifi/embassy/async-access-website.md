# Connecting ESP32 to Wi-Fi and Accessing Websites Asynchronously with Embassy

In this exercise, we will once again use the STA mode to access a website. But this time, we will do it asynchronously. We will use the reqwless crate as our HTTP client and the Embassy framework to enable asynchronous capabilities for embedded environments. Along with these, we will include additional helper crates.


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
embassy-net = { version = "0.7.1", features = [
  "defmt",
  "dhcpv4",
  "medium-ethernet",
  "tcp",
  "udp",
  #addition:
  "dns",
] }
```

### smoltcp

The smoltcp crate also gets added to the Cargo.toml. We need to add feature "dns-max-server-count-4" in order to use DNS servers.

```toml
# Updated smoltcp in Cargo.toml
smoltcp = { version = "0.12.0", default-features = false, features = [
    "defmt",
    "medium-ethernet",
    "multicast",
    "proto-dhcpv4",
    "proto-dns",
    "proto-ipv4",
    "socket-dns",
    "socket-icmp",
    "socket-raw",
    "socket-tcp",
    "socket-udp",
    # addition:
    "dns-max-server-count-4", 
] }
```

### Reqwless
The reqwless crate is an HTTP client for embedded systems, working with any transport that implements traits from the embedded-io crate.

```toml
reqwless = { version = "0.13.0", default-features = false, features = [
    "embedded-tls",
] }
```
