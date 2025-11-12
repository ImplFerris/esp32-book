# Notifier


This task demonstrates how to push data to a connected central device. Imagine it as sensor data that you periodically gather and send to a mobile device or some other device. In this example, we're simulating sensor readings by incrementing a counter every 2 seconds and sending notifications to the connected device.


```rust
/// Example task to use the BLE notifier interface.
/// This task will notify the connected central of a counter value every 2 seconds.
/// It will also read the RSSI value every 2 seconds.
/// and will stop when the connection is closed by the central or an error occurs.
async fn custom_task<C: Controller, P: PacketPool>(
    server: &Server<'_>,
    conn: &GattConnection<'_, '_, P>,
    stack: &Stack<'_, C, P>,
) {
    let mut tick: u8 = 0;
    let sensor_data = server.sensor_service.sensor_data;
    loop {
        tick = tick.wrapping_add(1);
        info!("[custom_task] notifying connection of tick {}", tick);
        if sensor_data.notify(conn, &tick).await.is_err() {
            info!("[custom_task] error notifying connection");
            break;
        };
        // read RSSI (Received Signal Strength Indicator) of the connection.
        if let Ok(rssi) = conn.raw().rssi(stack).await {
            info!("[custom_task] RSSI: {:?}", rssi);
        } else {
            info!("[custom_task] error getting RSSI");
            break;
        };
        Timer::after_secs(2).await;
    }
}
```

## Sending Notifications

```rust
let mut tick: u8 = 0;
let sensor_data = server.sensor_service.sensor_data;
loop {
    tick = tick.wrapping_add(1);
    info!("[custom_task] notifying connection of tick {}", tick);
    if sensor_data.notify(conn, &tick).await.is_err() {
        info!("[custom_task] error notifying connection");
        break;
    };
    // ...
    Timer::after_secs(2).await;
}
```

We maintain a counter that increments every loop iteration. The `notify()` method sends the current counter value to the connected central device. If the notification fails (usually because the connection dropped), we exit the loop.

## Reading Signal Strength
```rust
if let Ok(rssi) = conn.raw().rssi(stack).await {
    info!("[custom_task] RSSI: {:?}", rssi);
} else {
    info!("[custom_task] error getting RSSI");
    break;
};
```

RSSI (Received Signal Strength Indicator) measures the connection quality in dBm. We read this value to monitor how strong the connection is. If reading the RSSI fails, it typically indicates the connection has been lost, so we exit the task.
