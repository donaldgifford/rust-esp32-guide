# Lab 4 Solutions

<!--toc:start-->
- [Lab 4 Solutions](#lab-4-solutions)
  - [Solution 4.1: Health Endpoint](#solution-41-health-endpoint)
  - [Solution 4.2: LED Control via POST Endpoint](#solution-42-led-control-via-post-endpoint)
<!--toc:end-->

[Back to Part 4: Hands-On Labs](./README.md)

## Solution 4.1: Health Endpoint

**Exercise:** Add a `/health` endpoint that returns system status.

**Approach:** A `/health` endpoint is standard practice for any networked service — Kubernetes probes, load balancers, and monitoring systems all expect one. On an ESP32, "system status" means heap memory, uptime, WiFi signal strength (RSSI), and the chip's internal state. We return JSON to keep it compatible with monitoring tools.

**Key concepts:**

- **`esp_get_free_heap_size()`** — Returns available heap in bytes. On ESP32, you typically start with ~300KB and need to watch for leaks since there's no swap.
- **`esp_timer_get_time()`** — Microseconds since boot, from a hardware timer. Dividing to get seconds for human-readable uptime.
- **`unsafe` blocks** — These ESP-IDF sys calls are FFI bindings to C functions, hence `unsafe`. They're well-tested and safe to call — the `unsafe` is a Rust formality for FFI, not a warning about the functions themselves.

This snippet shows just the handler to add to the Lab 4 server. Register it after the existing `/` and `/stats` handlers:

```rust
// Add this handler alongside the existing "/" and "/stats" handlers in main()
server.fn_handler("/health", embedded_svc::http::Method::Get, move |req| {
    let free_heap = unsafe { esp_idf_svc::sys::esp_get_free_heap_size() };
    let uptime_us = unsafe { esp_idf_svc::sys::esp_timer_get_time() };
    let uptime_secs = uptime_us / 1_000_000;

    let json = format!(
        r#"{{"status":"ok","free_heap_bytes":{},"uptime_seconds":{},"firmware":"lab4"}}"#,
        free_heap, uptime_secs
    );

    let mut response = req.into_response(
        200,
        Some("OK"),
        &[("Content-Type", "application/json")],
    )?;
    response.write_all(json.as_bytes())?;
    Ok(())
})?;
```

**Full main.rs** with the health endpoint integrated:

```rust
use embedded_svc::io::Write;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::http::server::{Configuration as HttpConfig, EspHttpServer};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::info;
use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::Arc;

const WIFI_SSID: &str = "YourNetworkName";
const WIFI_PASS: &str = "YourPassword";

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 4 Exercise 4.1: Health Endpoint");

    let peripherals = Peripherals::take()?;
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sys_loop.clone(), Some(nvs))?,
        sys_loop,
    )?;

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: WIFI_SSID.try_into().unwrap(),
        password: WIFI_PASS.try_into().unwrap(),
        auth_method: AuthMethod::WPA2Personal,
        ..Default::default()
    }))?;

    wifi.start()?;
    wifi.connect()?;
    wifi.wait_netif_up()?;

    let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
    info!("Server at http://{}", ip_info.ip);

    let request_count = Arc::new(AtomicU32::new(0));
    let mut server = EspHttpServer::new(&HttpConfig::default())?;

    // Root endpoint (same as lab)
    let count_clone = request_count.clone();
    server.fn_handler("/", embedded_svc::http::Method::Get, move |req| {
        let count = count_clone.fetch_add(1, Ordering::SeqCst) + 1;
        let html = format!(
            r#"<!DOCTYPE html>
<html><head><title>ESP32</title></head>
<body>
  <h1>Hello from ESP32!</h1>
  <p>Requests: {}</p>
  <p><a href="/stats">Stats</a> | <a href="/health">Health</a></p>
</body></html>"#,
            count
        );
        req.into_ok_response()?.write_all(html.as_bytes())?;
        Ok(())
    })?;

    // Stats endpoint (same as lab)
    let count_clone = request_count.clone();
    server.fn_handler("/stats", embedded_svc::http::Method::Get, move |req| {
        let count = count_clone.load(Ordering::SeqCst);
        let free_heap = unsafe { esp_idf_svc::sys::esp_get_free_heap_size() };
        let json = format!(
            r#"{{"requests":{},"free_heap_bytes":{},"uptime_ms":{}}}"#,
            count,
            free_heap,
            unsafe { esp_idf_svc::sys::esp_timer_get_time() / 1000 }
        );
        let mut resp = req.into_response(200, Some("OK"), &[("Content-Type", "application/json")])?;
        resp.write_all(json.as_bytes())?;
        Ok(())
    })?;

    // Health endpoint
    server.fn_handler("/health", embedded_svc::http::Method::Get, move |req| {
        let free_heap = unsafe { esp_idf_svc::sys::esp_get_free_heap_size() };
        let uptime_secs = unsafe { esp_idf_svc::sys::esp_timer_get_time() } / 1_000_000;

        let json = format!(
            r#"{{"status":"ok","free_heap_bytes":{},"uptime_seconds":{},"firmware":"lab4"}}"#,
            free_heap, uptime_secs
        );

        let mut resp = req.into_response(200, Some("OK"), &[("Content-Type", "application/json")])?;
        resp.write_all(json.as_bytes())?;
        Ok(())
    })?;

    info!("Endpoints:");
    info!("  http://{}/", ip_info.ip);
    info!("  http://{}/stats", ip_info.ip);
    info!("  http://{}/health", ip_info.ip);

    loop {
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}
```

---

## Solution 4.2: LED Control via POST Endpoint

**Exercise:** Implement a `/led` POST endpoint that accepts `{"state": "on"}` or `{"state": "off"}`.

**Approach:** This exercise combines HTTP request parsing with GPIO output — bridging the network and hardware worlds. The main challenge is that the HTTP handler closure needs to own (or share) the LED pin driver, but `PinDriver` isn't `Clone`. We solve this with `Arc<Mutex<PinDriver>>`, which gives shared ownership with interior mutability.

**Why not just parse JSON with serde?** On an ESP32 with limited heap, pulling in `serde` + `serde_json` adds significant binary size (~50-100KB). For a simple two-value payload, manual string matching is pragmatic. In the final projects (Part 5), where JSON payloads are more complex, serde becomes worthwhile.

**Key concepts:**

- **`Arc<Mutex<PinDriver>>`** — The HTTP server runs handlers on its own thread pool. The LED pin must be shared across the main thread and handler closures. `Arc` provides shared ownership, `Mutex` provides exclusive access.
- **Reading POST body** — `req.read(&mut buf)` fills a buffer with the request body. We convert to a string and do simple substring matching.
- **HTTP method routing** — The same path can have different handlers for GET vs POST. We register a GET handler to return current state and a POST handler to change it.

```rust
use embedded_svc::io::{Read, Write};
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::http::server::{Configuration as HttpConfig, EspHttpServer};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::info;
use std::sync::{Arc, Mutex};

const WIFI_SSID: &str = "YourNetworkName";
const WIFI_PASS: &str = "YourPassword";

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 4 Exercise 4.2: LED POST Endpoint");

    let peripherals = Peripherals::take()?;
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    // Wrap the LED in Arc<Mutex<>> so HTTP handlers can share it
    let led = PinDriver::output(peripherals.pins.gpio8)?;
    let led = Arc::new(Mutex::new(led));

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sys_loop.clone(), Some(nvs))?,
        sys_loop,
    )?;

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: WIFI_SSID.try_into().unwrap(),
        password: WIFI_PASS.try_into().unwrap(),
        auth_method: AuthMethod::WPA2Personal,
        ..Default::default()
    }))?;

    wifi.start()?;
    wifi.connect()?;
    wifi.wait_netif_up()?;

    let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
    info!("Server at http://{}", ip_info.ip);

    let mut server = EspHttpServer::new(&HttpConfig::default())?;

    // GET /led — return current LED state
    let led_clone = led.clone();
    server.fn_handler("/led", embedded_svc::http::Method::Get, move |req| {
        let led = led_clone.lock().unwrap();
        let state = if led.is_set_high() { "on" } else { "off" };
        let json = format!(r#"{{"state":"{}"}}"#, state);

        let mut resp = req.into_response(200, Some("OK"), &[("Content-Type", "application/json")])?;
        resp.write_all(json.as_bytes())?;
        Ok(())
    })?;

    // POST /led — set LED state from {"state": "on"} or {"state": "off"}
    let led_clone = led.clone();
    server.fn_handler("/led", embedded_svc::http::Method::Post, move |mut req| {
        // Read the request body
        let mut buf = [0u8; 128];
        let bytes_read = req.read(&mut buf)?;
        let body = std::str::from_utf8(&buf[..bytes_read]).unwrap_or("");

        info!("POST /led body: {}", body);

        // Simple string matching — avoids pulling in serde for a two-value payload
        let mut led = led_clone.lock().unwrap();
        let (status, message) = if body.contains(r#""on""#) {
            led.set_high().ok();
            (200, r#"{"state":"on"}"#)
        } else if body.contains(r#""off""#) {
            led.set_low().ok();
            (200, r#"{"state":"off"}"#)
        } else {
            (400, r#"{"error":"expected {\"state\": \"on\"} or {\"state\": \"off\"}"}"#)
        };

        let mut resp = req.into_response(status, None, &[("Content-Type", "application/json")])?;
        resp.write_all(message.as_bytes())?;
        Ok(())
    })?;

    info!("Endpoints:");
    info!("  GET  http://{}/led    — read LED state", ip_info.ip);
    info!("  POST http://{}/led    — set LED state", ip_info.ip);
    info!("");
    info!("Test with:");
    info!(r#"  curl -X POST http://{}/led -d '{{"state":"on"}}'"#, ip_info.ip);
    info!(r#"  curl -X POST http://{}/led -d '{{"state":"off"}}'"#, ip_info.ip);

    loop {
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}
```

**Testing it:**

```bash
# Check current state
curl http://<ESP32_IP>/led

# Turn LED on
curl -X POST http://<ESP32_IP>/led -d '{"state":"on"}'

# Turn LED off
curl -X POST http://<ESP32_IP>/led -d '{"state":"off"}'

# Bad request
curl -X POST http://<ESP32_IP>/led -d '{"state":"maybe"}'
# Returns: {"error":"expected {\"state\": \"on\"} or {\"state\": \"off\"}"}
```
