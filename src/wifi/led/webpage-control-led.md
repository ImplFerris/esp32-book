# Control ESP32 LED with Webpage

We will create a simple API endpoint that accepts a boolean input to control the LED's state. Along with this, an "index.html" page will be served, displaying two buttons: one to turn the LED on and another to turn it off.

When you press one of the buttons, a request will be sent to the "/led" endpoint with the following JSON payload:

- `{ "is_on": true }` to turn the LED on
- `{ "is_on": false }` to turn the LED off

Based on the value of the "is_on" field, the "LED_STATE" variable of the led module will be updated. The "led_task" will then turn the LED on or off accordingly.

## Routing

In the "build_app" function, we configure the web routes for the application. The root path ("/") will serve the "index.html" content, we have to place this file inside "src/" folder. The "/led" path will accept "POST" requests and be handled by the "led_handler".

```rust
pub struct Application;

impl AppBuilder for Application {
    type PathRouter = impl routing::PathRouter;

    fn build_app(self) -> picoserve::Router<Self::PathRouter> {
        picoserve::Router::new()
            .route(
                "/",
                routing::get_service(File::html(include_str!("index.html"))),
            )
            .route("/led", routing::post(led_handler))
    }
}
```

## LED Handler

We will define two structs, one for handling the incoming input and one for sending the response. The LedRequest struct will derive Deserialize to parse the incoming JSON and provide it as a struct instance. The LedResponse struct will derive Serialize to convert the struct instance and send it as a JSON response.

```rust
#[derive(serde::Deserialize)]
struct LedRequest {
    is_on: bool,
}

#[derive(serde::Serialize)]
struct LedResponse {
    success: bool,
}
```

In the led_handler function, the LedRequest is extracted as a parameter. We can directly store the "is_on" value in the LED_STATE since both are boolean. Finally, the handler will return a JSON response with a LedResponse indicating success.

```rust
async fn led_handler(input: picoserve::extract::Json<LedRequest>) -> impl IntoResponse {
    crate::led::LED_STATE.store(input.0.is_on, Ordering::Relaxed);

    picoserve::response::Json(LedResponse { success: true })
}
```


## WebPage content

You can download the index.html file from [here](https://github.com/ImplFerris/esp32-projects/blob/main/wifi-led/src/index.html) and place it in the "src/" folder, or create your own custom content to send JSON requests.

**NOTE:**

You need to update the URL "http://192.168.0.50/led" with your ESP32's IP address. I've hardcoded it here for simplicity; otherwise, we would need to use a placeholder and replace it dynamically or adopt a template-based approach.

```html
<div class="button-container">
        <button class="btn-on" onclick="sendRequest(true)">Turn on LED</button>
        <button class="btn-off" onclick="sendRequest(false)">Turn off LED</button>
    </div>

    <script>
        function sendRequest(is_on) {
            const url = 'http://192.168.0.50/led'; // Replace with STATIC IP of ESP32

            fetch(url, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ is_on })
            })
            .then(response => {
                if (response.ok) {
                    return response.json();
                }
                throw new Error('Network response was not ok');
            })
            .then(data => {
                console.log('Success:', data);
                //alert(LED turned ${action});
            })
            .catch(error => {
                console.error('Error:', error);
                alert('Failed to send the request');
            });
        }
    </script>
```



## Clone the existing project
You can also clone (or refer) project I created and navigate to the `wifi-led` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/wifi-led
```

### How to run?

Pass the Wi-Fi name, password, static IP, and gateway IP address as environment variables, then flash the ESP32

```sh
SSID=YOUR_WIFI_NAME PASSWORD=YOUR_WIFI_PASSWORD STATIC_IP=ASSIGN_ESP32_IP/24 GATEWAY_IP=WIFI_GATEWAY_IP  cargo run --release
```
