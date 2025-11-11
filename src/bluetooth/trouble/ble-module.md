# BLE Module

In this module, we will define what kind of data our ESP32 sends and receives, set up and run the server, and handle connections. We've taken this example from the official Trouble repository and made some small changes. You can find more examples in the [repository](https://github.com/embassy-rs/trouble/tree/main/examples/apps).

Let's create the BLE module by updating the lib.rs file:

```rust
#![no_std]
pub mod ble;
```

From now on, we will be working in the `ble.rs` module file. Let's add all the necessary imports first:

```rust
use defmt::{info, warn};

use embassy_futures::join::join;
use embassy_futures::select::select;

use embassy_time::Timer;

// BLE:
use trouble_host::prelude::*;
```

Let's define the maximum number of concurrent connections and L2CAP channels for the BLE stack:

```rust
/// Max number of connections
const CONNECTIONS_MAX: usize = 1;

/// Max number of L2CAP channels.
const L2CAP_CHANNELS_MAX: usize = 1;
```


## GATT Service

We will use the gatt_server and gatt_service macros to define a GATT server with a sensor service containing two characteristics.

```rust
// GATT Server definition
#[gatt_server]
struct Server {
    sensor_service: SensorService,
}

/// Battery service
#[gatt_service(uuid = "a9c81b72-0f7a-4c59-b0a8-425e3bcf0a0e")]
struct SensorService {
    #[characteristic(uuid = "13c0ef83-09bd-4767-97cb-ee46224ae6db", read, notify)]
    sensor_data: u8,

    #[characteristic(uuid = "c79b2ca7-f39d-4060-8168-816fa26737b7", write, read)]
    sensor_settings: bool,
}
```

The Server struct represents our GATT server and contains a SensorService. The SensorService defines two characteristics:

- sensor_data: An "u8" value that mimics sensor data for this exercise. It supports read and notify operations, allowing clients to read the current sensor value and receive notifications when it changes. We will later update this value in a loop to demonstrate real-time updates.  Once you learn how to use BLE, you can also try using it with a real sensor and send data to your mobile device.

- sensor_settings: A boolean value that supports both write and read operations, enabling clients to configure sensor settings and read the current configuration. This is to show that we can modify the value from the client.
