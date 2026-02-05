# Part 5: Final Projects

<!--toc:start-->
- [Part 5: Final Projects](#part-5-final-projects)
  - [Project A: Home Assistant Sensor Integration](#project-a-home-assistant-sensor-integration)
  - [Project B: Prometheus Metrics Exporter](#project-b-prometheus-metrics-exporter)
<!--toc:end-->

## Project A: Home Assistant Sensor Integration

**Goal:** Build a complete sensor node that reports temperature and system stats to Home Assistant via MQTT discovery.

**Features:**

- Internal temperature sensor reading
- System stats (heap, uptime)
- Home Assistant MQTT auto-discovery
- LED status indicator
- Configurable update interval

**Create new project:**

```bash
cargo generate esp-rs/esp-idf-template cargo
# Name: ha-sensor-node
# MCU: esp32c6
```

**Full src/main.rs:**

```rust
use embedded_svc::mqtt::client::QoS;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::hal::temp_sensor::{TempSensorConfig, TempSensorDriver};
use esp_idf_svc::mqtt::client::{EspMqttClient, MqttClientConfiguration};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::{info, warn, error};
use std::time::Duration;

// Configuration
const WIFI_SSID: &str = "YourNetworkName";
const WIFI_PASS: &str = "YourPassword";
const MQTT_BROKER: &str = "mqtt://homeassistant.local:1883";
const MQTT_USER: Option<&str> = Some("mqtt_user");      // Set if required
const MQTT_PASS: Option<&str> = Some("mqtt_password");  // Set if required

const DEVICE_ID: &str = "esp32c6_sensor_01";
const UPDATE_INTERVAL_SECS: u64 = 30;

// MQTT Topics
const DISCOVERY_PREFIX: &str = "homeassistant";
const STATE_TOPIC: &str = "esp32/sensor_01/state";

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("=== Home Assistant Sensor Node ===");
    info!("Device ID: {}", DEVICE_ID);

    let peripherals = Peripherals::take()?;
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    // Initialize LED for status indication
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;
    led.set_low()?;

    // Initialize internal temperature sensor
    let temp_sensor = TempSensorDriver::new(&TempSensorConfig::default())?;
    info!("Temperature sensor initialized");

    // Connect to WiFi
    info!("Connecting to WiFi...");
    led_blink(&mut led, 2, 100)?;

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

    let mut retry_count = 0;
    while retry_count < 10 {
        match wifi.connect() {
            Ok(_) => break,
            Err(e) => {
                warn!("WiFi connect attempt {} failed: {:?}", retry_count + 1, e);
                retry_count += 1;
                FreeRtos::delay_ms(2000);
            }
        }
    }

    wifi.wait_netif_up()?;
    let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
    info!("WiFi connected! IP: {}", ip_info.ip);

    led_blink(&mut led, 3, 200)?;

    // Connect to MQTT
    info!("Connecting to MQTT broker: {}", MQTT_BROKER);

    let mqtt_config = MqttClientConfiguration {
        client_id: Some(DEVICE_ID),
        username: MQTT_USER,
        password: MQTT_PASS,
        ..Default::default()
    };

    let (mut client, mut connection) = EspMqttClient::new(MQTT_BROKER, &mqtt_config)?;

    // Handle MQTT events in background
    std::thread::spawn(move || {
        while let Ok(event) = connection.next() {
            info!("MQTT: {:?}", event.payload());
        }
    });

    FreeRtos::delay_ms(1000);

    // Publish Home Assistant discovery configs
    publish_discovery_configs(&mut client)?;
    info!("Published HA discovery configs");

    led.set_high()?;  // Solid LED = connected and running

    // Main sensor loop
    let mut iteration: u64 = 0;
    loop {
        iteration += 1;

        // Read sensors
        let temperature = temp_sensor.get_celsius().unwrap_or(-999.0);
        let free_heap = unsafe { esp_idf_svc::sys::esp_get_free_heap_size() };
        let uptime_ms = unsafe { esp_idf_svc::sys::esp_timer_get_time() / 1000 } as u64;

        // Create state payload
        let state_json = format!(
            r#"{{"temperature":{:.1},"free_heap":{},"uptime_sec":{},"wifi_rssi":{},"iteration":{}}}"#,
            temperature,
            free_heap,
            uptime_ms / 1000,
            get_wifi_rssi(&wifi),
            iteration
        );

        // Publish state
        match client.publish(STATE_TOPIC, QoS::AtLeastOnce, false, state_json.as_bytes()) {
            Ok(_) => {
                info!("Published: {}", state_json);
                led_blink(&mut led, 1, 50)?;
            }
            Err(e) => {
                error!("Publish failed: {:?}", e);
                led_blink(&mut led, 5, 100)?;
            }
        }

        // Check WiFi connection
        if !wifi.is_connected().unwrap_or(false) {
            warn!("WiFi disconnected, reconnecting...");
            led.set_low()?;
            if let Err(e) = wifi.connect() {
                error!("Reconnect failed: {:?}", e);
            }
        }

        std::thread::sleep(Duration::from_secs(UPDATE_INTERVAL_SECS));
    }
}

fn publish_discovery_configs(client: &mut EspMqttClient<'_>) -> anyhow::Result<()> {
    let device_info = format!(
        r#""device":{{"identifiers":["{}"],"name":"ESP32-C6 Sensor","manufacturer":"Espressif","model":"ESP32-C6-DevKitC-1"}}"#,
        DEVICE_ID
    );

    // Temperature sensor discovery
    let temp_config = format!(
        r#"{{"name":"Temperature","unique_id":"{}_temp","state_topic":"{}","device_class":"temperature","unit_of_measurement":"°C","value_template":"{{{{ value_json.temperature }}}}",{}}}"#,
        DEVICE_ID, STATE_TOPIC, device_info
    );

    client.publish(
        &format!("{}/sensor/{}_temp/config", DISCOVERY_PREFIX, DEVICE_ID),
        QoS::AtLeastOnce,
        true,  // retained
        temp_config.as_bytes(),
    )?;

    // Free heap sensor discovery
    let heap_config = format!(
        r#"{{"name":"Free Heap","unique_id":"{}_heap","state_topic":"{}","unit_of_measurement":"bytes","value_template":"{{{{ value_json.free_heap }}}}","entity_category":"diagnostic",{}}}"#,
        DEVICE_ID, STATE_TOPIC, device_info
    );

    client.publish(
        &format!("{}/sensor/{}_heap/config", DISCOVERY_PREFIX, DEVICE_ID),
        QoS::AtLeastOnce,
        true,
        heap_config.as_bytes(),
    )?;

    // WiFi RSSI sensor discovery
    let rssi_config = format!(
        r#"{{"name":"WiFi Signal","unique_id":"{}_rssi","state_topic":"{}","device_class":"signal_strength","unit_of_measurement":"dBm","value_template":"{{{{ value_json.wifi_rssi }}}}","entity_category":"diagnostic",{}}}"#,
        DEVICE_ID, STATE_TOPIC, device_info
    );

    client.publish(
        &format!("{}/sensor/{}_rssi/config", DISCOVERY_PREFIX, DEVICE_ID),
        QoS::AtLeastOnce,
        true,
        rssi_config.as_bytes(),
    )?;

    // Uptime sensor discovery
    let uptime_config = format!(
        r#"{{"name":"Uptime","unique_id":"{}_uptime","state_topic":"{}","device_class":"duration","unit_of_measurement":"s","value_template":"{{{{ value_json.uptime_sec }}}}","entity_category":"diagnostic",{}}}"#,
        DEVICE_ID, STATE_TOPIC, device_info
    );

    client.publish(
        &format!("{}/sensor/{}_uptime/config", DISCOVERY_PREFIX, DEVICE_ID),
        QoS::AtLeastOnce,
        true,
        uptime_config.as_bytes(),
    )?;

    Ok(())
}

fn get_wifi_rssi(wifi: &BlockingWifi<EspWifi<'_>>) -> i8 {
    wifi.wifi()
        .sta_netif()
        .get_ip_info()
        .ok()
        .map(|_| {
            // Note: Getting actual RSSI requires scanning or specific API
            // This is a placeholder - in real code, use wifi.scan() or specific RSSI API
            -50i8
        })
        .unwrap_or(-100)
}

fn led_blink<P: OutputPin>(led: &mut PinDriver<'_, P, Output>, times: u8, ms: u32) -> anyhow::Result<()> {
    for _ in 0..times {
        led.set_high()?;
        FreeRtos::delay_ms(ms);
        led.set_low()?;
        FreeRtos::delay_ms(ms);
    }
    Ok(())
}
```

**Home Assistant configuration.yaml (if not using auto-discovery):**

```yaml
mqtt:
  sensor:
    - name: "ESP32 Temperature"
      state_topic: "esp32/sensor_01/state"
      value_template: "{{ value_json.temperature }}"
      unit_of_measurement: "°C"
      device_class: temperature

    - name: "ESP32 Free Heap"
      state_topic: "esp32/sensor_01/state"
      value_template: "{{ value_json.free_heap }}"
      unit_of_measurement: "bytes"
```

---

## Project B: Prometheus Metrics Exporter

**Goal:** Build a Prometheus-compatible metrics endpoint that can be scraped for monitoring.

**Create new project:**

```bash
cargo generate esp-rs/esp-idf-template cargo
# Name: prometheus-exporter
# MCU: esp32c6
```

**Full src/main.rs:**

```rust
use embedded_svc::http::server::Method;
use embedded_svc::io::Write;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::hal::temp_sensor::{TempSensorConfig, TempSensorDriver};
use esp_idf_svc::http::server::{Configuration as HttpConfig, EspHttpServer};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::info;
use std::sync::atomic::{AtomicU32, AtomicU64, Ordering};
use std::sync::Arc;
use std::time::Duration;

const WIFI_SSID: &str = "YourNetworkName";
const WIFI_PASS: &str = "YourPassword";
const HTTP_PORT: u16 = 9100;  // Standard node_exporter port

// Metrics storage
struct Metrics {
    http_requests_total: AtomicU64,
    temperature_celsius: AtomicU32,  // Store as i32 * 100 for precision
    free_heap_bytes: AtomicU32,
    wifi_rssi_dbm: AtomicU32,  // Store as i32 + 128 to make unsigned
    uptime_seconds: AtomicU64,
    boot_count: AtomicU32,
}

impl Metrics {
    fn new() -> Self {
        Self {
            http_requests_total: AtomicU64::new(0),
            temperature_celsius: AtomicU32::new(0),
            free_heap_bytes: AtomicU32::new(0),
            wifi_rssi_dbm: AtomicU32::new(128),  // -128 + 128 = 0 as default
            uptime_seconds: AtomicU64::new(0),
            boot_count: AtomicU32::new(1),
        }
    }

    fn set_temperature(&self, temp: f32) {
        self.temperature_celsius.store((temp * 100.0) as u32, Ordering::Relaxed);
    }

    fn get_temperature(&self) -> f32 {
        self.temperature_celsius.load(Ordering::Relaxed) as f32 / 100.0
    }

    fn set_rssi(&self, rssi: i8) {
        self.wifi_rssi_dbm.store((rssi as i32 + 128) as u32, Ordering::Relaxed);
    }

    fn get_rssi(&self) -> i8 {
        (self.wifi_rssi_dbm.load(Ordering::Relaxed) as i32 - 128) as i8
    }

    fn to_prometheus_format(&self, device_id: &str) -> String {
        let requests = self.http_requests_total.load(Ordering::Relaxed);
        let temperature = self.get_temperature();
        let heap = self.free_heap_bytes.load(Ordering::Relaxed);
        let rssi = self.get_rssi();
        let uptime = self.uptime_seconds.load(Ordering::Relaxed);
        let boots = self.boot_count.load(Ordering::Relaxed);

        format!(
            r#"# HELP esp32_temperature_celsius Internal temperature sensor reading
# TYPE esp32_temperature_celsius gauge
esp32_temperature_celsius{{device="{device_id}"}} {temperature:.2}

# HELP esp32_free_heap_bytes Available heap memory in bytes
# TYPE esp32_free_heap_bytes gauge
esp32_free_heap_bytes{{device="{device_id}"}} {heap}

# HELP esp32_wifi_rssi_dbm WiFi signal strength in dBm
# TYPE esp32_wifi_rssi_dbm gauge
esp32_wifi_rssi_dbm{{device="{device_id}"}} {rssi}

# HELP esp32_uptime_seconds Seconds since boot
# TYPE esp32_uptime_seconds counter
esp32_uptime_seconds{{device="{device_id}"}} {uptime}

# HELP esp32_boot_count_total Number of times the device has booted
# TYPE esp32_boot_count_total counter
esp32_boot_count_total{{device="{device_id}"}} {boots}

# HELP esp32_http_requests_total Total HTTP requests to /metrics
# TYPE esp32_http_requests_total counter
esp32_http_requests_total{{device="{device_id}"}} {requests}
"#,
            device_id = device_id,
            temperature = temperature,
            heap = heap,
            rssi = rssi,
            uptime = uptime,
            boots = boots,
            requests = requests,
        )
    }
}

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("=== Prometheus Metrics Exporter ===");

    let peripherals = Peripherals::take()?;
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    // Initialize LED
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;
    led.set_low()?;

    // Initialize temperature sensor
    let temp_sensor = TempSensorDriver::new(&TempSensorConfig::default())?;

    // Initialize shared metrics
    let metrics = Arc::new(Metrics::new());

    // Get boot count from NVS (persists across reboots)
    // In a full implementation, you'd read/write this to NVS

    // Connect to WiFi
    info!("Connecting to WiFi...");
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
    info!("WiFi connected! IP: {}", ip_info.ip);

    // Create HTTP server
    let server_config = HttpConfig {
        http_port: HTTP_PORT,
        ..Default::default()
    };

    let mut server = EspHttpServer::new(&server_config)?;

    // /metrics endpoint (Prometheus scrape target)
    let metrics_clone = metrics.clone();
    server.fn_handler("/metrics", Method::Get, move |req| {
        metrics_clone.http_requests_total.fetch_add(1, Ordering::Relaxed);

        let body = metrics_clone.to_prometheus_format("esp32c6_01");

        let mut response = req.into_response(
            200,
            Some("OK"),
            &[
                ("Content-Type", "text/plain; version=0.0.4; charset=utf-8"),
            ],
        )?;
        response.write_all(body.as_bytes())?;
        Ok(())
    })?;

    // / endpoint (info page)
    server.fn_handler("/", Method::Get, move |req| {
        let html = r#"<!DOCTYPE html>
<html>
<head><title>ESP32 Prometheus Exporter</title></head>
<body>
    <h1>ESP32 Prometheus Exporter</h1>
    <p><a href="/metrics">Metrics</a></p>
</body>
</html>"#;

        req.into_ok_response()?.write_all(html.as_bytes())?;
        Ok(())
    })?;

    // /health endpoint (for Prometheus blackbox or k8s probes)
    server.fn_handler("/health", Method::Get, move |req| {
        req.into_ok_response()?.write_all(b"OK")?;
        Ok(())
    })?;

    info!("HTTP server running on port {}", HTTP_PORT);
    info!("  http://{}:{}/", ip_info.ip, HTTP_PORT);
    info!("  http://{}:{}/metrics", ip_info.ip, HTTP_PORT);

    led.set_high()?;

    // Background task: update metrics
    let metrics_updater = metrics.clone();
    loop {
        // Update temperature
        if let Ok(temp) = temp_sensor.get_celsius() {
            metrics_updater.set_temperature(temp);
        }

        // Update heap
        let free_heap = unsafe { esp_idf_svc::sys::esp_get_free_heap_size() };
        metrics_updater.free_heap_bytes.store(free_heap, Ordering::Relaxed);

        // Update uptime
        let uptime_us = unsafe { esp_idf_svc::sys::esp_timer_get_time() };
        metrics_updater.uptime_seconds.store((uptime_us / 1_000_000) as u64, Ordering::Relaxed);

        // Update RSSI (simplified - actual implementation would use wifi scan)
        metrics_updater.set_rssi(-55);

        // Blink LED to show activity
        led.set_low()?;
        FreeRtos::delay_ms(50);
        led.set_high()?;

        std::thread::sleep(Duration::from_secs(5));
    }
}
```

**Prometheus scrape config (prometheus.yml):**

```yaml
scrape_configs:
  - job_name: 'esp32'
    static_configs:
      - targets: ['192.168.1.XXX:9100']  # Your ESP32's IP
    scrape_interval: 30s
    scrape_timeout: 10s
```

**Grafana dashboard query examples:**

```promql
# Temperature over time
esp32_temperature_celsius{device="esp32c6_01"}

# Memory usage percentage (assuming 320KB total)
(327680 - esp32_free_heap_bytes{device="esp32c6_01"}) / 327680 * 100

# Request rate
rate(esp32_http_requests_total{device="esp32c6_01"}[5m])

# Uptime in hours
esp32_uptime_seconds{device="esp32c6_01"} / 3600
```
