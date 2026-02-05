# Part 4: Hands-On Labs

<!--toc:start-->
- [Part 4: Hands-On Labs](#part-4-hands-on-labs)
  - [Lab 1: Hello World and LED Blink](#lab-1-hello-world-and-led-blink)
  - [Lab 2: Reading Digital Inputs (Button Press)](#lab-2-reading-digital-inputs-button-press)
  - [Lab 3: WiFi Connection](#lab-3-wifi-connection)
  - [Lab 4: HTTP Server](#lab-4-http-server)
  - [Lab 5: Reading Analog Sensors (ADC)](#lab-5-reading-analog-sensors-adc)
  - [Lab 6: MQTT Client (Home Assistant Preparation)](#lab-6-mqtt-client-home-assistant-preparation)
<!--toc:end-->

## Lab 1: Hello World and LED Blink

**Objective:** Verify your environment and understand the basic project lifecycle.

**Hardware Setup:**

- Connect your ESP32-C6 or ESP32-S3 via USB
- The onboard LED is typically on GPIO8 (C6) or GPIO48 (S3)

**Create the project:**

```bash
cargo generate esp-rs/esp-idf-template cargo
# Name: lab1-blink
# MCU: esp32c6 (or esp32s3)
```

**Replace src/main.rs:**

```rust
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use log::info;

fn main() -> anyhow::Result<()> {
    // Initialize ESP-IDF
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 1: LED Blink");

    // Get peripheral access
    let peripherals = Peripherals::take()?;

    // Configure the LED pin as output
    // ESP32-C6 onboard LED: GPIO8
    // ESP32-S3 onboard LED: GPIO48 (RGB, but we'll treat as simple)
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;

    info!("Starting blink loop...");

    loop {
        // Turn LED on
        led.set_high()?;
        info!("LED ON");
        FreeRtos::delay_ms(500);

        // Turn LED off
        led.set_low()?;
        info!("LED OFF");
        FreeRtos::delay_ms(500);
    }
}
```

**Build and flash:**

```bash
cd lab1-blink
cargo run
```

**Exercise 1.1:** Modify the blink pattern to blink SOS in Morse code (... --- ...).

**Exercise 1.2:** Add a variable `blink_count` and log how many times the LED has toggled.

**[Solutions for Lab 1](lab1-solutions.md)**

---

## Lab 2: Reading Digital Inputs (Button Press)

**Objective:** Learn GPIO input handling and understand polling vs. interrupts.

**Hardware Setup:**

- Connect a push button between GPIO9 and GND (using internal pull-up)
- Or use the BOOT button (GPIO9 on most dev boards)

**Replace src/main.rs:**

```rust
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use log::info;

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 2: Button Input");

    let peripherals = Peripherals::take()?;

    // Configure LED as output
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;

    // Configure button as input with internal pull-up
    // When button is pressed, it connects to GND = LOW
    let button = PinDriver::input(peripherals.pins.gpio9)?;

    // Track button state for edge detection
    let mut last_state = button.is_high();
    let mut press_count: u32 = 0;

    info!("Press the BOOT button...");

    loop {
        let current_state = button.is_high();

        // Detect falling edge (button press)
        if last_state && !current_state {
            press_count += 1;
            info!("Button pressed! Count: {}", press_count);
            led.set_high()?;
        }

        // Detect rising edge (button release)
        if !last_state && current_state {
            info!("Button released");
            led.set_low()?;
        }

        last_state = current_state;

        // Small delay to debounce
        FreeRtos::delay_ms(10);
    }
}
```

**Exercise 2.1:** Implement a "long press" detector that triggers after holding for 2 seconds.

**Exercise 2.2:** Create a mode toggle—short press cycles through 3 LED blink speeds.

**[Solutions for Lab 2](lab2-solutions.md)**

---

## Lab 3: WiFi Connection

**Objective:** Connect to a WiFi network and understand the ESP-IDF networking stack.

**Add to Cargo.toml:**

```toml
[dependencies]
esp-idf-svc = { version = "0.49", features = ["binstart"] }
esp-idf-hal = "0.44"
embedded-svc = "0.28"
log = "0.4"
anyhow = "1.0"
```

**Replace src/main.rs:**

```rust
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::{info, error};
use std::time::Duration;

// Replace with your WiFi credentials
const WIFI_SSID: &str = "YourNetworkName";
const WIFI_PASS: &str = "YourPassword";

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 3: WiFi Connection");

    // Initialize required components
    let peripherals = Peripherals::take()?;
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    // Create WiFi driver
    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sys_loop.clone(), Some(nvs))?,
        sys_loop,
    )?;

    // Configure as station (client)
    let wifi_config = Configuration::Client(ClientConfiguration {
        ssid: WIFI_SSID.try_into().unwrap(),
        password: WIFI_PASS.try_into().unwrap(),
        auth_method: AuthMethod::WPA2Personal,
        ..Default::default()
    });

    wifi.set_configuration(&wifi_config)?;

    info!("Starting WiFi...");
    wifi.start()?;

    info!("Scanning for networks...");
    let scan_result = wifi.scan()?;
    for ap in scan_result.iter().take(10) {
        info!("Found AP: {} (signal: {}dBm)", ap.ssid, ap.signal_strength);
    }

    info!("Connecting to {}...", WIFI_SSID);
    match wifi.connect() {
        Ok(_) => info!("Connected to WiFi!"),
        Err(e) => {
            error!("Failed to connect: {:?}", e);
            return Err(e.into());
        }
    }

    info!("Waiting for IP address...");
    wifi.wait_netif_up()?;

    let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
    info!("IP address: {:?}", ip_info.ip);
    info!("Gateway: {:?}", ip_info.subnet.gateway);
    info!("Subnet: {:?}", ip_info.subnet.mask);

    info!("WiFi connected successfully!");

    // Keep the program running
    loop {
        // Check if still connected
        if !wifi.is_connected()? {
            error!("WiFi disconnected, attempting reconnect...");
            wifi.connect()?;
        }
        std::thread::sleep(Duration::from_secs(10));
    }
}
```

**Exercise 3.1:** Implement automatic reconnection with exponential backoff.

**Exercise 3.2:** Store WiFi credentials in NVS (non-volatile storage) so they survive reboots.

**[Solutions for Lab 3](lab3-solutions.md)**

---

## Lab 4: HTTP Server

**Objective:** Create a basic HTTP server to serve data from the ESP32.

**Replace src/main.rs:**

```rust
use embedded_svc::http::server::{Handler, HandlerResult, Request};
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

    info!("Lab 4: HTTP Server");

    // Initialize WiFi (same as Lab 3)
    let peripherals = Peripherals::take()?;
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sys_loop.clone(), Some(nvs))?,
        sys_loop,
    )?;

    let wifi_config = Configuration::Client(ClientConfiguration {
        ssid: WIFI_SSID.try_into().unwrap(),
        password: WIFI_PASS.try_into().unwrap(),
        auth_method: AuthMethod::WPA2Personal,
        ..Default::default()
    });

    wifi.set_configuration(&wifi_config)?;
    wifi.start()?;
    wifi.connect()?;
    wifi.wait_netif_up()?;

    let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
    info!("Server starting at http://{}", ip_info.ip);

    // Shared state: request counter
    let request_count = Arc::new(AtomicU32::new(0));

    // Create HTTP server
    let mut server = EspHttpServer::new(&HttpConfig::default())?;

    // Root endpoint
    let count_clone = request_count.clone();
    server.fn_handler("/", embedded_svc::http::Method::Get, move |req| {
        let count = count_clone.fetch_add(1, Ordering::SeqCst) + 1;

        let html = format!(
            r#"<!DOCTYPE html>
<html>
<head><title>ESP32 Server</title></head>
<body>
    <h1>Hello from ESP32!</h1>
    <p>Request count: {}</p>
    <p><a href="/stats">View Stats JSON</a></p>
</body>
</html>"#,
            count
        );

        req.into_ok_response()?
            .write_all(html.as_bytes())?;
        Ok(())
    })?;

    // JSON stats endpoint
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

        let mut response = req.into_response(
            200,
            Some("OK"),
            &[("Content-Type", "application/json")],
        )?;
        response.write_all(json.as_bytes())?;
        Ok(())
    })?;

    info!("HTTP server running!");
    info!("  http://{}/", ip_info.ip);
    info!("  http://{}/stats", ip_info.ip);

    // Keep server running
    loop {
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}
```

**Exercise 4.1:** Add a `/health` endpoint that returns system status.

**Exercise 4.2:** Implement a `/led` POST endpoint that accepts `{"state": "on"}` or `{"state": "off"}`.

**[Solutions for Lab 4](lab4-solutions.md)**

---

## Lab 5: Reading Analog Sensors (ADC)

**Objective:** Read analog values from sensors like temperature or light sensors.

**Hardware Setup:**

- Connect a potentiometer or analog sensor to GPIO1 (ADC1 channel)
- Or use the internal temperature sensor (no external hardware needed)

```rust
use esp_idf_svc::hal::adc::{attenuation, AdcChannelDriver, AdcDriver};
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::prelude::*;
use log::info;

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 5: ADC Reading");

    let peripherals = Peripherals::take()?;

    // Configure ADC
    let mut adc = AdcDriver::new(peripherals.adc1)?;

    // Configure ADC channel on GPIO1 with 11dB attenuation (0-3.3V range)
    let mut adc_pin: AdcChannelDriver<'_, { attenuation::DB_11 }, _> =
        AdcChannelDriver::new(peripherals.pins.gpio1)?;

    loop {
        // Read raw ADC value (12-bit: 0-4095)
        let raw_value = adc.read(&mut adc_pin)?;

        // Convert to voltage (assuming 3.3V reference)
        let voltage = (raw_value as f32 / 4095.0) * 3.3;

        // If using a thermistor or LM35, convert to temperature
        // LM35: 10mV per degree C
        let temperature_c = voltage * 100.0;

        info!(
            "ADC Raw: {}, Voltage: {:.3}V, Estimated Temp: {:.1}°C",
            raw_value, voltage, temperature_c
        );

        FreeRtos::delay_ms(1000);
    }
}
```

**For ESP32-C6 Internal Temperature Sensor:**

```rust
use esp_idf_svc::hal::temp_sensor::{TempSensorConfig, TempSensorDriver};

let temp_sensor = TempSensorDriver::new(&TempSensorConfig::default())?;
let temperature = temp_sensor.get_celsius()?;
info!("Internal temp: {:.1}°C", temperature);
```

---

## Lab 6: MQTT Client (Home Assistant Preparation)

**Objective:** Publish sensor data via MQTT, the protocol Home Assistant uses for IoT integration.

**Add to Cargo.toml:**

```toml
[dependencies]
esp-idf-svc = { version = "0.49", features = ["binstart"] }
embedded-svc = "0.28"
log = "0.4"
anyhow = "1.0"
```

```rust
use embedded_svc::mqtt::client::{Connection, MessageImpl, QoS};
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::mqtt::client::{EspMqttClient, MqttClientConfiguration};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::{info, error};
use std::time::Duration;

const WIFI_SSID: &str = "YourNetworkName";
const WIFI_PASS: &str = "YourPassword";
const MQTT_BROKER: &str = "mqtt://192.168.1.100:1883";  // Your MQTT broker
const MQTT_CLIENT_ID: &str = "esp32-sensor";
const MQTT_TOPIC: &str = "homeassistant/sensor/esp32/state";

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 6: MQTT Client");

    // Initialize WiFi (abbreviated)
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
    info!("WiFi connected!");

    // Configure MQTT client
    let mqtt_config = MqttClientConfiguration {
        client_id: Some(MQTT_CLIENT_ID),
        ..Default::default()
    };

    // Create MQTT client with callback for incoming messages
    let (mut client, mut connection) = EspMqttClient::new(MQTT_BROKER, &mqtt_config)?;

    // Spawn a thread to handle MQTT events
    std::thread::spawn(move || {
        info!("MQTT event handler started");
        while let Ok(event) = connection.next() {
            info!("MQTT Event: {:?}", event.payload());
        }
        info!("MQTT connection closed");
    });

    // Give time for connection
    std::thread::sleep(Duration::from_secs(2));

    info!("Publishing sensor data...");

    let mut reading = 0;
    loop {
        // Simulate sensor reading
        reading += 1;
        let temperature = 20.0 + (reading as f32 * 0.1).sin() * 5.0;
        let humidity = 50.0 + (reading as f32 * 0.05).cos() * 10.0;

        // Home Assistant compatible JSON
        let payload = format!(
            r#"{{"temperature":{:.1},"humidity":{:.1},"uptime":{}}}"#,
            temperature, humidity, reading
        );

        match client.publish(MQTT_TOPIC, QoS::AtLeastOnce, false, payload.as_bytes()) {
            Ok(_) => info!("Published: {}", payload),
            Err(e) => error!("Publish failed: {:?}", e),
        }

        std::thread::sleep(Duration::from_secs(10));
    }
}
```
