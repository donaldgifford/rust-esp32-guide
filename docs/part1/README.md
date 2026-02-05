# Part 1: Environment Setup

<!--toc:start-->
- [Part 1: Environment Setup](#part-1-environment-setup)
  - [1.1 Understanding the ESP32 Ecosystem](#11-understanding-the-esp32-ecosystem)
  - [1.2 Installing the Rust Toolchain](#12-installing-the-rust-toolchain)
  - [1.3 Installing ESP32 Rust Tooling](#13-installing-esp32-rust-tooling)
  - [1.4 Installing ESP-IDF Prerequisites](#14-installing-esp-idf-prerequisites)
  - [1.5 USB Driver Setup for ARM Mac](#15-usb-driver-setup-for-arm-mac)
  - [1.6 Verifying Your Setup](#16-verifying-your-setup)
<!--toc:end-->

## 1.1 Understanding the ESP32 Ecosystem

Before diving in, let's map familiar concepts to the embedded world:

| Cloud/Server Concept | Embedded Equivalent |
|---------------------|---------------------|
| OS Kernel | Bare metal or RTOS (FreeRTOS) |
| systemd services | Main loop or RTOS tasks |
| Package manager | Cargo (with embedded crates) |
| Docker containers | Firmware images |
| kubectl apply | espflash flash |
| Prometheus endpoint | HTTP server on microcontroller |
| MQTT broker | Same, but you're the client |

The ESP32 runs either bare-metal (no OS) or on FreeRTOS. Espressif provides `esp-idf` (their C SDK) and the community maintains `esp-rs` for Rust support. We'll use `esp-idf-hal` which wraps esp-idf, giving us WiFi, Bluetooth, and full peripheral access.

## 1.2 Installing the Rust Toolchain

```bash
# Install Rust via rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Restart your shell or source the env
source "$HOME/.cargo/env"

# Verify installation
rustc --version
cargo --version
```

## 1.3 Installing ESP32 Rust Tooling

The ESP32-C6 uses RISC-V (supported by mainline Rust), while the ESP32-S3 uses Xtensa (requires Espressif's fork). We'll install tooling for both.

```bash
# Install espup - manages ESP Rust toolchains
cargo install espup

# Install the ESP Rust toolchain (includes Xtensa support)
espup install

# Source the export file (add to your .zshrc for persistence)
source "$HOME/export-esp.sh"

# Install additional tools
cargo install cargo-espflash    # For flashing firmware
cargo install espflash          # Standalone flasher
cargo install ldproxy           # Linker proxy for esp-idf
cargo install cargo-generate    # Project templates
```

## 1.4 Installing ESP-IDF Prerequisites

```bash
# Install system dependencies via Homebrew
brew install cmake ninja dfu-util python3

# Install the esp-idf Python dependencies
pip3 install --user esptool
```

## 1.5 USB Driver Setup for ARM Mac

Modern Macs with Apple Silicon work with ESP32 dev boards out of the box via the built-in USB CDC driver. However, some boards use different USB-to-serial chips:

```bash
# List connected USB devices
system_profiler SPUSBDataType

# When you plug in your ESP32, you should see it as a serial device
ls /dev/cu.usb*
```

**Expected device paths:**

- ESP32-C6-DevKitC-1: `/dev/cu.usbmodem*` (built-in USB-JTAG)
- ESP32-S3-WROOM-1: `/dev/cu.usbserial*` or `/dev/cu.usbmodem*`

## 1.6 Verifying Your Setup

Let's create a minimal test project:

```bash
# Create a new project from the esp-idf template
cargo generate esp-rs/esp-idf-template cargo

# When prompted:
# - Project name: hello-esp32
# - MCU: esp32c6 (or esp32s3)
# - Configure advanced template options: false
```

```bash
cd hello-esp32

# Build the project
cargo build

# You should see successful compilation with output like:
# Compiling hello-esp32 v0.1.0
# Finished dev [optimized + debuginfo] target(s)
```

If this builds successfully, your environment is ready.
