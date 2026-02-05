# Rust ESP32 Quick Reference

A companion reference for the Rust ESP32 Development Guide. Keep this handy for quick lookups during development.

---

## Command Cheatsheet

### Project Setup

```bash
# Create new project
cargo generate esp-rs/esp-idf-template cargo

# Build
cargo build
cargo build --release

# Flash and monitor
cargo run
cargo espflash flash --monitor
cargo espflash flash --port /dev/cu.usbmodem14101 --monitor

# Monitor only
espflash monitor
espflash monitor --port /dev/cu.usbmodem14101

# Clean build
cargo clean
rm -rf target/

# Update dependencies
cargo update
```

### Environment Management

```bash
# Source ESP toolchain (add to .zshrc)
source ~/export-esp.sh

# Check toolchain
rustup show
espup --version

# Reinstall toolchain
espup uninstall
espup install
```

### Serial Port Commands

```bash
# List USB devices (macOS)
ls /dev/cu.usb*
system_profiler SPUSBDataType

# Find ESP32 port
ls /dev/cu.usb* | grep -E 'usbserial|usbmodem'

# Reset device manually
# Hold BOOT → Press RESET → Release BOOT → Release RESET
```

---

## GPIO Pin Maps

### ESP32-C6-DevKitC-1

| Function | GPIO | Notes |
|----------|------|-------|
| Onboard LED | 8 | Active high |
| BOOT Button | 9 | Active low, internal pull-up |
| USB D- | 12 | USB-JTAG |
| USB D+ | 13 | USB-JTAG |
| ADC1_CH0 | 0 | Analog input |
| ADC1_CH1 | 1 | Analog input |
| ADC1_CH2 | 2 | Analog input |
| ADC1_CH3 | 3 | Analog input |
| ADC1_CH4 | 4 | Analog input |
| I2C SDA | 6 | Default |
| I2C SCL | 7 | Default |
| SPI MOSI | 7 | Default |
| SPI MISO | 2 | Default |
| SPI CLK | 6 | Default |
| UART TX | 16 | Default |
| UART RX | 17 | Default |

### ESP32-S3-WROOM-1

| Function | GPIO | Notes |
|----------|------|-------|
| Onboard RGB LED | 48 | WS2812 (addressable) |
| BOOT Button | 0 | Active low |
| USB D- | 19 | Native USB |
| USB D+ | 20 | Native USB |
| ADC1_CH0 | 1 | Analog input |
| ADC1_CH1 | 2 | Analog input |
| I2C SDA | 8 | Default |
| I2C SCL | 9 | Default |
| SPI MOSI | 11 | Default |
| SPI MISO | 13 | Default |
| SPI CLK | 12 | Default |
| UART TX | 43 | Default |
| UART RX | 44 | Default |

---

## Cargo.toml Templates

### Minimal Project

```toml
[package]
name = "my-esp32-project"
version = "0.1.0"
edition = "2021"

[dependencies]
esp-idf-svc = "0.49"
esp-idf-hal = "0.44"
log = "0.4"
anyhow = "1.0"

[build-dependencies]
embuild = "0.32"
```

### Full-Featured Project

```toml
[package]
name = "my-esp32-project"
version = "0.1.0"
edition = "2021"

[dependencies]
esp-idf-svc = { version = "0.49", features = ["binstart"] }
esp-idf-hal = "0.44"
embedded-svc = "0.28"
log = "0.4"
anyhow = "1.0"

# Serialization
serde = { version = "1.0", default-features = false, features = ["derive"] }
serde_json = "1.0"

# Time handling
chrono = { version = "0.4", default-features = false }

# HTTP client
embedded-svc = "0.28"

[build-dependencies]
embuild = "0.32"

[profile.release]
opt-level = "s"       # Optimize for size
lto = true            # Link-time optimization
debug = false

[profile.dev]
opt-level = 1         # Some optimization even in dev
debug = true
```

---

## Code Snippets

### Project Initialization

```rust
fn main() -> anyhow::Result<()> {
    // MUST be first - initializes ESP-IDF
    esp_idf_svc::sys::link_patches();
    
    // Initialize logging
    esp_idf_svc::log::EspLogger::initialize_default();
    
    // Take peripherals (can only be done once)
    let peripherals = esp_idf_svc::hal::prelude::Peripherals::take()?;
    
    // Your code here...
    
    Ok(())
}
```

### GPIO Output (LED)

```rust
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;

let mut led = PinDriver::output(peripherals.pins.gpio8)?;

led.set_high()?;   // Turn on
led.set_low()?;    // Turn off
led.toggle()?;     // Toggle state

// Check current state
if led.is_set_high() {
    log::info!("LED is on");
}
```

### GPIO Input (Button)

```rust
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;

// Input with internal pull-up
let button = PinDriver::input(peripherals.pins.gpio9)?;

// Read state
if button.is_low() {
    log::info!("Button pressed");
}

// With pull-up/pull-down configuration
let mut button = PinDriver::input(peripherals.pins.gpio9)?;
button.set_pull(Pull::Up)?;
```

### GPIO Interrupt

```rust
use esp_idf_svc::hal::gpio::*;
use std::sync::atomic::{AtomicBool, Ordering};

static BUTTON_PRESSED: AtomicBool = AtomicBool::new(false);

let mut button = PinDriver::input(peripherals.pins.gpio9)?;
button.set_pull(Pull::Up)?;
button.set_interrupt_type(InterruptType::NegEdge)?;

unsafe {
    button.subscribe(|| {
        BUTTON_PRESSED.store(true, Ordering::SeqCst);
    })?;
}

button.enable_interrupt()?;

loop {
    if BUTTON_PRESSED.swap(false, Ordering::SeqCst) {
        log::info!("Button interrupt triggered!");
    }
    FreeRtos::delay_ms(10);
}
```

### ADC Reading

```rust
use esp_idf_svc::hal::adc::{attenuation, AdcChannelDriver, AdcDriver};

let mut adc = AdcDriver::new(peripherals.adc1)?;
let mut adc_pin: AdcChannelDriver<'_, { attenuation::DB_11 }, _> =
    AdcChannelDriver::new(peripherals.pins.gpio1)?;

// Read raw value (0-4095 for 12-bit)
let raw = adc.read(&mut adc_pin)?;

// Convert to voltage (3.3V reference with 11dB attenuation)
let voltage = (raw as f32 / 4095.0) * 3.3;
```

### Internal Temperature Sensor

```rust
use esp_idf_svc::hal::temp_sensor::{TempSensorConfig, TempSensorDriver};

let temp_sensor = TempSensorDriver::new(&TempSensorConfig::default())?;
let temperature = temp_sensor.get_celsius()?;
log::info!("Temperature: {:.1}°C", temperature);
```

### WiFi Connection

```rust
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};

let sys_loop = EspSystemEventLoop::take()?;
let nvs = EspDefaultNvsPartition::take()?;

let mut wifi = BlockingWifi::wrap(
    EspWifi::new(peripherals.modem, sys_loop.clone(), Some(nvs))?,
    sys_loop,
)?;

wifi.set_configuration(&Configuration::Client(ClientConfiguration {
    ssid: "MyNetwork".try_into().unwrap(),
    password: "MyPassword".try_into().unwrap(),
    auth_method: AuthMethod::WPA2Personal,
    ..Default::default()
}))?;

wifi.start()?;
wifi.connect()?;
wifi.wait_netif_up()?;

let ip = wifi.wifi().sta_netif().get_ip_info()?.ip;
log::info!("IP: {}", ip);
```

### HTTP Server

```rust
use embedded_svc::http::server::Method;
use embedded_svc::io::Write;
use esp_idf_svc::http::server::{Configuration, EspHttpServer};

let mut server = EspHttpServer::new(&Configuration::default())?;

server.fn_handler("/", Method::Get, |req| {
    req.into_ok_response()?.write_all(b"Hello!")?;
    Ok(())
})?;

server.fn_handler("/json", Method::Get, |req| {
    let json = r#"{"status":"ok"}"#;
    let mut resp = req.into_response(200, Some("OK"), &[
        ("Content-Type", "application/json"),
    ])?;
    resp.write_all(json.as_bytes())?;
    Ok(())
})?;
```

### HTTP Client

```rust
use embedded_svc::http::client::Client;
use embedded_svc::io::Read;
use esp_idf_svc::http::client::EspHttpConnection;

let mut client = Client::wrap(EspHttpConnection::new(&Default::default())?);

let request = client.get("http://httpbin.org/ip")?;
let mut response = request.submit()?;

let mut body = [0u8; 1024];
let bytes_read = response.read(&mut body)?;
let body_str = std::str::from_utf8(&body[..bytes_read])?;
log::info!("Response: {}", body_str);
```

### MQTT Client

```rust
use embedded_svc::mqtt::client::QoS;
use esp_idf_svc::mqtt::client::{EspMqttClient, MqttClientConfiguration};

let config = MqttClientConfiguration {
    client_id: Some("esp32-client"),
    username: Some("user"),
    password: Some("pass"),
    ..Default::default()
};

let (mut client, mut connection) = EspMqttClient::new("mqtt://broker:1883", &config)?;

// Handle events in background thread
std::thread::spawn(move || {
    while let Ok(event) = connection.next() {
        log::info!("MQTT: {:?}", event.payload());
    }
});

// Publish
client.publish("topic/test", QoS::AtLeastOnce, false, b"Hello MQTT")?;

// Subscribe
client.subscribe("topic/commands", QoS::AtLeastOnce)?;
```

### Delay and Timing

```rust
use esp_idf_svc::hal::delay::FreeRtos;
use std::time::{Duration, Instant};

// FreeRTOS delay (yields to other tasks)
FreeRtos::delay_ms(100);

// Standard library sleep
std::thread::sleep(Duration::from_millis(100));

// Get uptime
let uptime_us = unsafe { esp_idf_svc::sys::esp_timer_get_time() };
let uptime_sec = uptime_us / 1_000_000;

// Measure duration
let start = Instant::now();
// ... do something
let elapsed = start.elapsed();
```

### System Information

```rust
// Free heap memory
let free_heap = unsafe { esp_idf_svc::sys::esp_get_free_heap_size() };
let min_free_heap = unsafe { esp_idf_svc::sys::esp_get_minimum_free_heap_size() };

// Chip info
let chip_info = unsafe {
    let mut info = std::mem::MaybeUninit::uninit();
    esp_idf_svc::sys::esp_chip_info(info.as_mut_ptr());
    info.assume_init()
};
log::info!("Cores: {}", chip_info.cores);

// MAC address
let mut mac = [0u8; 6];
unsafe {
    esp_idf_svc::sys::esp_read_mac(mac.as_mut_ptr(), esp_idf_svc::sys::esp_mac_type_t_ESP_MAC_WIFI_STA);
}
log::info!("MAC: {:02X}:{:02X}:{:02X}:{:02X}:{:02X}:{:02X}", 
    mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
```

### NVS (Non-Volatile Storage)

```rust
use esp_idf_svc::nvs::{EspNvs, EspNvsPartition, NvsDefault};

let nvs_partition = EspNvsPartition::<NvsDefault>::take()?;
let mut nvs = EspNvs::new(nvs_partition, "storage", true)?;

// Write values
nvs.set_u32("boot_count", 42)?;
nvs.set_str("wifi_ssid", "MyNetwork")?;

// Read values
let count = nvs.get_u32("boot_count")?.unwrap_or(0);
let mut ssid_buf = [0u8; 32];
let ssid = nvs.get_str("wifi_ssid", &mut ssid_buf)?.unwrap_or("");
```

---

## Rust Syntax Quick Reference

### Ownership Patterns

```rust
// Move (transfer ownership)
let s1 = String::from("hello");
let s2 = s1;  // s1 is now invalid

// Clone (copy data)
let s1 = String::from("hello");
let s2 = s1.clone();  // Both valid

// Borrow (reference)
let s1 = String::from("hello");
process(&s1);  // Borrow
println!("{}", s1);  // Still valid

// Mutable borrow
let mut s1 = String::from("hello");
modify(&mut s1);  // Mutable borrow
```

### Result Handling

```rust
// Match
match do_something() {
    Ok(value) => println!("Got: {}", value),
    Err(e) => println!("Error: {:?}", e),
}

// ? operator (propagate error)
fn my_function() -> Result<(), Error> {
    let value = do_something()?;  // Returns early on Err
    Ok(())
}

// unwrap (panic on error - avoid in production)
let value = do_something().unwrap();

// unwrap_or (default value)
let value = do_something().unwrap_or(default);

// unwrap_or_else (lazy default)
let value = do_something().unwrap_or_else(|e| {
    log::error!("Error: {:?}", e);
    default
});

// map (transform Ok value)
let doubled = result.map(|x| x * 2);

// and_then (chain operations)
let final_result = step1()
    .and_then(|x| step2(x))
    .and_then(|x| step3(x));
```

### Option Handling

```rust
// Match
match maybe_value {
    Some(v) => println!("Got: {}", v),
    None => println!("Nothing"),
}

// if let (when you only care about Some)
if let Some(v) = maybe_value {
    println!("Got: {}", v);
}

// unwrap_or
let value = maybe_value.unwrap_or(default);

// map
let doubled = maybe_value.map(|x| x * 2);

// ok_or (convert to Result)
let result: Result<i32, &str> = maybe_value.ok_or("was None");
```

### Iterators

```rust
let numbers = vec![1, 2, 3, 4, 5];

// Map (transform each element)
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();

// Filter
let evens: Vec<&i32> = numbers.iter().filter(|x| *x % 2 == 0).collect();

// Find (first match)
let first_even = numbers.iter().find(|x| *x % 2 == 0);

// Sum
let total: i32 = numbers.iter().sum();

// Any/All
let has_even = numbers.iter().any(|x| x % 2 == 0);
let all_positive = numbers.iter().all(|x| *x > 0);

// Enumerate
for (i, n) in numbers.iter().enumerate() {
    println!("[{}]: {}", i, n);
}

// Chunks
for chunk in numbers.chunks(2) {
    println!("{:?}", chunk);
}
```

### String Operations

```rust
// Create
let s = String::from("hello");
let s = "hello".to_string();
let s: String = format!("Value: {}", 42);

// Concatenate
let s = format!("{} {}", "hello", "world");
let s = "hello".to_string() + " world";

// Slice
let slice = &s[0..5];

// Contains
if s.contains("ell") { }

// Split
for part in s.split(',') { }

// Parse
let n: i32 = "42".parse().unwrap();
```

---

## Error Messages and Solutions

### Build Errors

| Error | Solution |
|-------|----------|
| `linker 'ldproxy' not found` | `cargo install ldproxy` |
| `esp-idf-sys build failed` | `source ~/export-esp.sh` |
| `target may not be installed` | `espup install` |
| `could not find esp-idf` | Check `$IDF_PATH`, reinstall with `espup` |
| `multiple versions of crate` | `cargo update` or check version conflicts |

### Runtime Errors

| Error | Solution |
|-------|----------|
| `panic - already borrowed` | Check for conflicting mutable borrows |
| `guru meditation error` | Stack overflow - increase stack size or reduce allocation |
| `wifi: sta disconnected` | Check SSID/password, signal strength |
| `mqtt: connection refused` | Check broker address, port, credentials |
| `out of memory` | Reduce buffers, avoid clones, use references |

### Flash Errors

| Error | Solution |
|-------|----------|
| `connection failed` | Hold BOOT, press RESET, release both |
| `device not found` | Check USB cable, try different port |
| `timeout` | Use slower baud rate: `--speed 115200` |
| `wrong chip type` | Check `target` in `.cargo/config.toml` |

---

## Performance Tips

### Memory Optimization

```rust
// Use stack allocation for fixed-size buffers
let mut buffer = [0u8; 256];  // Stack

// Avoid unnecessary heap allocation
// Bad:
let s = some_string.clone();
// Good (if possible):
let s = &some_string;

// Pre-allocate vectors if size is known
let mut v = Vec::with_capacity(100);

// Use references in function parameters
fn process(data: &[u8]) { }  // Takes slice, not Vec
```

### CPU Optimization

```rust
// Use release mode for production
// cargo build --release

// Add to Cargo.toml for size optimization:
[profile.release]
opt-level = "s"  # Optimize for size
lto = true       # Link-time optimization

// Avoid busy loops - use delays
loop {
    // Do work
    FreeRtos::delay_ms(10);  // Yield to other tasks
}
```

### Power Optimization

```rust
// Use light sleep for short waits
// (Implementation varies by ESP-IDF version)

// Use deep sleep for long periods
unsafe {
    esp_idf_svc::sys::esp_deep_sleep(1_000_000);  // 1 second
}

// Reduce WiFi power
// Configure in sdkconfig.defaults:
// CONFIG_ESP32_WIFI_DYNAMIC_TX_POWER=y
```

---

## sdkconfig.defaults Reference

```ini
# Logging
CONFIG_LOG_DEFAULT_LEVEL_INFO=y
# CONFIG_LOG_DEFAULT_LEVEL_DEBUG=y  # For debugging

# Stack sizes
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
CONFIG_PTHREAD_TASK_STACK_SIZE_DEFAULT=4096

# WiFi
CONFIG_ESP32_WIFI_STATIC_RX_BUFFER_NUM=10
CONFIG_ESP32_WIFI_DYNAMIC_RX_BUFFER_NUM=32
CONFIG_ESP_WIFI_SOFTAP_BEACON_MAX_LEN=752

# Networking
CONFIG_LWIP_MAX_SOCKETS=10
CONFIG_LWIP_SO_RCVBUF=y

# Power management
CONFIG_PM_ENABLE=y

# Partition table (for NVS, OTA, etc.)
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
```

---

## Useful Links

### Documentation

- [esp-rs Book](https://esp-rs.github.io/book/)
- [esp-idf-hal Docs](https://docs.rs/esp-idf-hal)
- [esp-idf-svc Docs](https://docs.rs/esp-idf-svc)
- [embedded-svc Docs](https://docs.rs/embedded-svc)

### Hardware

- [ESP32-C6 Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-c6_datasheet_en.pdf)
- [ESP32-C6 Technical Reference](https://www.espressif.com/sites/default/files/documentation/esp32-c6_technical_reference_manual_en.pdf)
- [ESP32-S3 Datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf)

### Community

- [esp-rs GitHub](https://github.com/esp-rs)
- [esp-rs Matrix Chat](https://matrix.to/#/#esp-rs:matrix.org)
- [Rust Embedded](https://github.com/rust-embedded)

### Tools

- [espflash](https://github.com/esp-rs/espflash)
- [espup](https://github.com/esp-rs/espup)
- [probe-rs](https://probe.rs/) (for debugging)
