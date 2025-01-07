# Final: Access the Website

We have covered how to set up the Wi-Fi connection with Embassy and walked through the overall flow leading up to the call to the access_website function. Since the previous chapter became quite lengthy, we didn't dive into the details of the access_website function. In this chapter, we will focus on how to send an HTTP request and receive a response from a website using the reqwless crate, and then we finally will run the code.


### Sending HTTP Request

We begin by initializing the DNS socket and the TCP client.

**TLS Config:**

Next, we set up the TLS configuration using the tls_seed, which is the random number we generated earlier and passed into the function. We configure TLS to skip SSL certificate verification by setting TlsVerify::None. While this is not recommended for production environments, we use this insecure setting for simplicity.  In production, you would need to configure TLS with certificate verification and provide the certificate chain so that it can use for verification. 

The remaining part is pretty straightforward. We initialize the HTTP client with the TCP client, DNS socket, and TLS config. Then, we send a GET request to the jsonplaceholder website, which returns JSON content. We read the content and then print it to the console.

```rust
async fn access_website(stack: &'static Stack<WifiDevice<'static, WifiStaDevice>>, tls_seed: u64) {
    let mut rx_buffer = [0; 4096];
    let mut tx_buffer = [0; 4096];
    let dns = DnsSocket::new(&stack);
    let tcp_state = TcpClientState::<1, 4096, 4096>::new();
    let tcp = TcpClient::new(stack, &tcp_state);

    let tls = TlsConfig::new(
        tls_seed,
        &mut rx_buffer,
        &mut tx_buffer,
        reqwless::client::TlsVerify::None,
    );

    let mut client = HttpClient::new_with_tls(&tcp, &dns, tls);
    let mut buffer = [0u8; 4096];
    let mut http_req = client
        .request(
            reqwless::request::Method::GET,
            "https://jsonplaceholder.typicode.com/posts/1",
        )
        .await
        .unwrap();
    let response = http_req.send(&mut buffer).await.unwrap();

    info!("Got response");
    let res = response.body().read_to_end().await.unwrap();

    let content = core::str::from_utf8(res).unwrap();
    println!("{}", content);
}
```

## Clone the existing project
You can also clone (or refer) project I created and navigate to the `wifi-async-http` folder.

```sh
git clone https://github.com/ImplFerris/esp32-projects
cd esp32-projects/wifi-async-http
```

### How to run?

Normally, we would simply run `cargo run --release`, but this time we also need to pass the environment variables for the Wi-Fi connection.

```rust
SSID=YOUR_WIFI_NAME PASSWORD=YOUR_WIFI_PASSWORD  cargo run --release
```
