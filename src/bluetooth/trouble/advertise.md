# BLE Advertising

BLE advertising is how a device announces its presence to nearby devices (like phones or computers) and shares information so they can connect. It broadcasts the device name, available services, and capabilities.

Once a central device initiates a connection, the function accepts it and returns a GATT connection instance that can be used to communicate with the connected device.

```rust
/// Create an advertiser to use to connect to a BLE Central, and wait for it to connect.
async fn advertise<'values, 'server, C: Controller>(
    name: &'values str,
    peripheral: &mut Peripheral<'values, C, DefaultPacketPool>,
    server: &'server Server<'values>,
) -> Result<GattConnection<'values, 'server, DefaultPacketPool>, BleHostError<C::Error>> {
    let mut advertiser_data = [0; 31];
    let len = AdStructure::encode_slice(
        &[
            AdStructure::Flags(LE_GENERAL_DISCOVERABLE | BR_EDR_NOT_SUPPORTED),
            AdStructure::CompleteLocalName(name.as_bytes()),
        ],
        &mut advertiser_data[..],
    )?;
    let advertiser = peripheral
        .advertise(
            &Default::default(),
            Advertisement::ConnectableScannableUndirected {
                adv_data: &advertiser_data[..len],
                scan_data: &[],
            },
        )
        .await?;
    info!("[adv] advertising");
    let conn = advertiser.accept().await?.with_attribute_server(server)?;
    info!("[adv] connection established");
    Ok(conn)
}
```

Let's break down this function and understand each step.

## Preparing Advertisement Data

```rust
let mut advertiser_data = [0; 31];
 
let len = AdStructure::encode_slice(
    &[
        AdStructure::Flags(LE_GENERAL_DISCOVERABLE | BR_EDR_NOT_SUPPORTED),
        AdStructure::CompleteLocalName(name.as_bytes()),
    ],
    &mut advertiser_data[..],
)?;
```

We create a buffer and encode two pieces of information:

- Flags: `LE_GENERAL_DISCOVERABLE` makes the device visible to all scanners, while `BR_EDR_NOT_SUPPORTED` indicates this is a BLE-only device (no classic Bluetooth).  You can find more details about these flags [here](../ble/gap.html#advertisement-flags).
- Device Name: The complete local name that appears when scanning for devices

The `encode_slice` function packs this data into the proper BLE advertising format and returns the actual length used.


## Starting Advertisement

```rust
let advertiser = peripheral
    .advertise(
        &Default::default(),
        Advertisement::ConnectableScannableUndirected {
            adv_data: &advertiser_data[..len],
            scan_data: &[],
        },
    )
    .await?;
```

We start advertising using `ConnectableScannableUndirected` mode, which means:
- Connectable: Central devices can connect to us
- Scannable: Central devices can request additional information (though we provide empty `scan_data` here)
- Undirected: We're broadcasting to everyone, not targeting a specific device

The device is now visible to nearby BLE scanners.


## Accepting Connections

```rust
let conn = advertiser.accept().await?.with_attribute_server(server)?;
Ok(conn)
```

The `accept()` call waits for a central device to initiate a connection. Once connected, we transform the connection into a `GattConnection` using `with_attribute_server()`, which attaches our GATT server (containing our services and characteristics) to the connection.
