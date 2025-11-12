# GATT Events

The GATT events task handles incoming requests from the connected central device. It processes read and write operations on characteristics and responds appropriately until the connection closes.

```rust
/// Stream Events until the connection closes.
///
/// This function will handle the GATT events and process them.
/// This is how we interact with read and write requests.
async fn gatt_events_task<P: PacketPool>(
    server: &Server<'_>,
    conn: &GattConnection<'_, '_, P>,
) -> Result<(), Error> {
    let sensor_data = server.sensor_service.sensor_data;

    let reason = loop {
        match conn.next().await {
            GattConnectionEvent::Disconnected { reason } => break reason,
            GattConnectionEvent::Gatt { event } => {
                match &event {
                    GattEvent::Read(event) => {
                        if event.handle() == sensor_data.handle {
                            let value = server.get(&sensor_data);
                            info!(
                                "[gatt] Read Event to Sensor Data Characteristic: {:?}",
                                value
                            );
                        }
                    }
                    GattEvent::Write(event) => {
                        if event.handle() == sensor_data.handle {
                            info!(
                                "[gatt] Write Event to Sensor Data Characteristic: {:?}",
                                event.data()
                            );
                        }
                    }
                    _ => {}
                };
                // This step is also performed at drop(), but writing it explicitly is necessary
                // in order to ensure reply is sent.
                match event.accept() {
                    Ok(reply) => reply.send().await,
                    Err(e) => warn!("[gatt] error sending response: {:?}", e),
                };
            }
            _ => {} // ignore other Gatt Connection Events
        }
    };

    info!("[gatt] disconnected: {:?}", reason);
    Ok(())
}
```

Let's break down this function and understand each step.

## Event Loop

```rust
let sensor_data = server.sensor_service.sensor_data;

let reason = loop {
    match conn.next().await {
        GattConnectionEvent::Disconnected { reason } => break reason,
        GattConnectionEvent::Gatt { event } => {
            // ... handle event
        }
        _ => {} // ignore other Gatt Connection Events
    }
};
info!("[gatt] disconnected: {:?}", reason);
```

We first get a reference to our sensor data characteristic, then enter a loop that waits for events on the connection. The loop continues until a disconnection event occurs, at which point it breaks and logs the disconnection reason.

## Handling Read Events

```rust
GattEvent::Read(event) => {
    if event.handle() == sensor_data.handle {
        let value = server.get(&sensor_data);
        info!(
            "[gatt] Read Event to Sensor Data Characteristic: {:?}",
            value
        );
    }
}
```

When a central device reads a characteristic, we check if the handle matches our sensor data characteristic. If it does, we retrieve the current value from the server and log it.  The library automatically handles sending the value when accept and send are called..

## Handling Write Events

```rust
GattEvent::Write(event) => {
    if event.handle() == sensor_data.handle {
        info!(
            "[gatt] Write Event to Sensor Data Characteristic: {:?}",
            event.data()
        );
    }
}
```

When a central device writes to a characteristic, we check if it's writing to our sensor data characteristic. If so, we log the data being written. The actual write will be processed when we call `accept()` on the event.


## Sending Responses

```rust
match event.accept() {
    Ok(reply) => reply.send().await,
    Err(e) => warn!("[gatt] error sending response: {:?}", e),
};
```

After processing each event, we must explicitly accept it and send a response back to the client. While this also happens automatically when the event is dropped, doing it explicitly ensures the response is sent immediately. If sending the response fails, we log a warning.
