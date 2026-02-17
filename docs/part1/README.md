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

# Source the export file — required in every new terminal
source "$HOME/export-esp.sh"

# Add to your shell config for persistence:
# echo 'source "$HOME/export-esp.sh"' >> ~/.zshrc
# Or add to ~/.zshenv if you want it available in non-interactive shells too

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

## 1.7 Common Setup Issues

### `can't find crate for core` / target not installed

```
error[E0463]: can't find crate for `core`
  = note: the `riscv32imac-esp-espidf` target may not be installed
```

This error is misleading — `rustup target add riscv32imac-esp-espidf` won't work because this isn't a standard rustup target. It's provided by the ESP toolchain.

**Check that `export-esp.sh` is sourced** in your current terminal:

```bash
source "$HOME/export-esp.sh"
```

**Check the active toolchain** inside the project directory:

```bash
cd hello-esp32
rustup show active-toolchain
```

It should show `esp` or `nightly` (depending on what `rust-toolchain.toml` specifies), not your default stable toolchain.

### Tool version managers (mise, asdf, rtx) overriding the toolchain

If you use `mise`, `asdf`, or similar tools to manage Rust versions, they will override `rust-toolchain.toml` and force a standard toolchain that doesn't know about ESP32 targets.

Check if mise is pinning Rust:

```bash
mise ls rust
```

If it shows an active Rust version for your project directory, remove it:

```bash
mise unset rust --local
```

Then verify the correct toolchain is active:

```bash
rustup show active-toolchain
# Should show "esp" or "nightly", not a standard version like "1.93.1-aarch64-apple-darwin"
```

### rust-analyzer not found for the `esp` toolchain

If your editor shows:

```
'rust-analyzer' is not installed for the custom toolchain 'esp'
```

The `esp` toolchain is custom and doesn't ship with rust-analyzer. Symlink it from stable:

```bash
ln -s ~/.rustup/toolchains/stable-aarch64-apple-darwin/bin/rust-analyzer \
      ~/.rustup/toolchains/esp/bin/rust-analyzer
```

Note: if your project's `rust-toolchain.toml` says `channel = "nightly"` rather than `channel = "esp"`, add rust-analyzer to nightly instead:

```bash
rustup component add rust-analyzer --toolchain nightly
```

### Project structure: don't use Cargo workspaces

Each ESP-IDF project is self-contained with its own `.cargo/config.toml` (build target), `build.rs`, and `sdkconfig.defaults`. Cargo workspaces don't inherit `.cargo/config.toml` from member directories, which breaks target selection and causes errors like:

```
Error: Unsupported target 'aarch64-apple-darwin'
```

Keep each lab as a separate project directory and open your editor in the specific project:

```bash
cd hello-esp32 && nvim .
```
