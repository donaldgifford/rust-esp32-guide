# Troubleshooting

<!--toc:start-->
- [Troubleshooting](#troubleshooting)
  - [Common Build Errors](#common-build-errors)
  - [Memory Issues](#memory-issues)
  - [Debugging Tips](#debugging-tips)
  - [Next Steps](#next-steps)
<!--toc:end-->

## Common Build Errors

**Error: `esp-idf-sys` build fails**

```bash
# Clean and rebuild
cargo clean
rm -rf target/
cargo build

# Ensure environment is sourced
source ~/export-esp.sh
```

**Error: `failed to run custom build command`**

```bash
# Check ESP-IDF installation
echo $IDF_PATH
# Should point to ~/.espup/esp-idf/...

# Reinstall toolchain
espup uninstall
espup install
source ~/export-esp.sh
```

**Error: Device not found when flashing**

```bash
# List serial devices
ls /dev/cu.usb*

# Try with explicit port
cargo espflash flash --port /dev/cu.usbmodem14101 --monitor

# Reset device: hold BOOT, press RESET, release BOOT
```

**Error: WiFi won't connect**

```rust
// Debug: Print scan results
let scan_result = wifi.scan()?;
for ap in scan_result {
    log::info!("AP: {} ({}dBm, {:?})", ap.ssid, ap.signal_strength, ap.auth_method);
}

// Common issues:
// - SSID/password typo
// - Wrong auth_method (try AuthMethod::WPAWPA2Personal)
// - 5GHz network (ESP32-C6 supports 5GHz, ESP32-S3 doesn't)
```

## Memory Issues

```rust
// Check heap usage
let free_heap = unsafe { esp_idf_svc::sys::esp_get_free_heap_size() };
let min_heap = unsafe { esp_idf_svc::sys::esp_get_minimum_free_heap_size() };
log::info!("Heap: {} free, {} minimum", free_heap, min_heap);

// If running low:
// - Reduce buffer sizes
// - Use stack allocation where possible
// - Avoid String::clone(), use references
```

## Debugging Tips

```bash
# Increase log verbosity in sdkconfig.defaults
CONFIG_LOG_DEFAULT_LEVEL_DEBUG=y

# Or set at runtime
esp_idf_svc::log::set_target_level("*", log::LevelFilter::Debug)?;

# Use espmonitor for better serial output
cargo install espmonitor
espmonitor /dev/cu.usbmodem14101
```

---

## Next Steps

After completing this guide, consider exploring:

1. **BLE (Bluetooth Low Energy)**: The ESP32-C6 and S3 both support BLE 5.0
2. **I2C/SPI Sensors**: Add BME280 (temp/humidity/pressure) or MPU6050 (accelerometer)
3. **OTA Updates**: Update firmware over WiFi without physical connection
4. **Deep Sleep**: Battery-powered applications with micro-amp sleep current
5. **Embassy**: Async Rust framework for embedded (alternative to esp-idf)
6. **Tauri + ESP**: Desktop app that communicates with your ESP32

**Useful Resources:**

- [esp-rs Book](https://esp-rs.github.io/book/)
- [esp-idf-hal Documentation](https://docs.rs/esp-idf-hal)
- [ESP32-C6 Technical Reference](https://www.espressif.com/en/products/socs/esp32-c6)
- [Rust Embedded Book](https://docs.rust-embedded.org/book/)
- [Home Assistant MQTT Discovery](https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery)
