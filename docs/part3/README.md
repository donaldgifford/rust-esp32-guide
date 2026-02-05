# Part 3: ESP32 Project Structure

## 3.1 Anatomy of an ESP32 Rust Project

```
hello-esp32/
├── .cargo/
│   └── config.toml      # Build settings, target architecture
├── src/
│   └── main.rs          # Your application code
├── build.rs             # Build script (generates bindings)
├── Cargo.toml           # Dependencies and metadata
├── sdkconfig.defaults   # ESP-IDF configuration
└── rust-toolchain.toml  # Rust version pinning
```

**Cargo.toml** - Key dependencies for ESP32 development:

```toml
[package]
name = "hello-esp32"
version = "0.1.0"
edition = "2021"

[dependencies]
esp-idf-svc = "0.49"      # High-level ESP-IDF services (WiFi, HTTP, etc.)
esp-idf-hal = "0.44"      # Hardware abstraction (GPIO, SPI, I2C, etc.)
embedded-svc = "0.28"     # Traits for embedded services
log = "0.4"               # Logging facade
anyhow = "1.0"            # Simplified error handling

[build-dependencies]
embuild = "0.32"          # ESP-IDF build integration
```

**.cargo/config.toml** - Target configuration:

```toml
[build]
target = "riscv32imac-esp-espidf"  # For ESP32-C6
# target = "xtensa-esp32s3-espidf" # For ESP32-S3

[target.riscv32imac-esp-espidf]
linker = "ldproxy"
runner = "espflash flash --monitor"

[env]
ESP_IDF_VERSION = "v5.2"  # ESP-IDF version to use
```

## 3.2 The Main Entry Point

```rust
// src/main.rs

use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::log::EspLogger;
use log::info;

fn main() -> anyhow::Result<()> {
    // Initialize ESP-IDF system
    esp_idf_svc::sys::link_patches();

    // Initialize logging
    EspLogger::initialize_default();

    info!("Starting application...");

    // Your code here

    loop {
        // Main loop
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}
```

## 3.3 Build and Flash Workflow

```bash
# Build only
cargo build

# Build and flash
cargo run

# Build, flash, and open serial monitor
cargo espflash flash --monitor

# Just open the serial monitor
espflash monitor
```

**Pro tip:** The serial monitor shows `println!` and `log::info!` output. Use `Ctrl+R` to reset the device, `Ctrl+C` to exit.
