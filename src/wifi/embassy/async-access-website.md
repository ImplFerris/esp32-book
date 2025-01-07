# Connecting ESP32 to Wi-Fi and Accessing Websites Asynchronously with Embassy

In this exercise, we will once again use the STA mode to access a website. But this time, we will do it asynchronously. We will use the reqwless crate as our HTTP client and the Embassy framework to enable asynchronous capabilities for embedded environments. Along with these, we will include additional helper crates.


## Generate project using esp-generate

To create the project, use the `esp-generate` command. Run the following:

```sh
esp-generate --chip esp32 wifi-async-http
```

This will open a screen asking you to select options. 

- Select the option "Enables Wi-Fi via the esp-wifi crate. Requires alloc".  It automatically selects the espa-alloc crate option also
- Select the option "Adds embassy framework support".

Just save it by pressing "s" in the keyboard.


## Update dependencies

### Embassy net

embassy-net, built on the smoltcp crate, provides an async network stack for embedded systems. It offers a higher-level API and supports IPv4, IPv6, Ethernet, bare-IP mediums, TCP, UDP, DNS, DHCPv4, and multicast. 

This crate automatically gets added when you select the Wi-Fi and Embassy option while generating the project with esp-generate. However, we need one additional feature: "dns". This is necessary to perform HTTP requests because the website name (e.g., google.com) needs to be resolved into an IP address. Remember, in the previous exercise, we manually provided the IP address of the website.

```toml
# Updated embassy-net in Cargo.toml
embassy-net = { version = "0.4.0", features = [
    "tcp",
    "udp",
    "dhcpv4",
    "medium-ethernet",
    #addition:
    "dns",
] }
```

### smoltcp
The smoltcp crate also gets added to the Cargo.toml. We need to add feature "dns-max-server-count-4" in order to use DNS servers.

```toml
# Updated smoltcp in Cargo.toml
smoltcp = { version = "0.11.0", default-features = false, features = [
    "medium-ethernet",
    "proto-dhcpv4",
    "proto-igmp",
    "proto-ipv4",
    "socket-dhcpv4",
    "socket-icmp",
    "socket-raw",
    "socket-tcp",
    "socket-udp",
    # addition:
    "dns-max-server-count-4", 
] }
```

### Embassy Executor
embassy-executor crate is an async/await executor specifically designed for embedded systems.

"When the nightly Cargo feature is not enabled, embassy-executor allocates tasks out of an arena (a very simple bump allocator)."  We can specify the arena size via environment settings or as a feature flag. When we enabled Embassy support, it added the `task-arena-size-12288` feature. But for our task, this won't be enough, so we will use the `task-arena-size-32768` feature instead. You can read more about this [here](https://docs.embassy.dev/embassy-executor/git/cortex-m/index.html#task-arena).

```toml
embassy-executor = { version = "0.6.0", features = ["task-arena-size-32768"] }
```

### Reqwless
The reqwless crate is an HTTP client for embedded systems, working with any transport that implements traits from the embedded-io crate.

```toml
reqwless = { version = "0.12.0", default-features = false, features = [
    "embedded-tls",
] }
```
