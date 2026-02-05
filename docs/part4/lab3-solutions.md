# Lab 3 Solutions

<!--toc:start-->
- [Lab 3 Solutions](#lab-3-solutions)
  - [Solution 3.1: Automatic Reconnection with Exponential Backoff](#solution-31-automatic-reconnection-with-exponential-backoff)
  - [Solution 3.2: Store WiFi Credentials in NVS](#solution-32-store-wifi-credentials-in-nvs)
<!--toc:end-->

[Back to Part 4: Hands-On Labs](./README.md)

## Solution 3.1: Automatic Reconnection with Exponential Backoff

**Exercise:** Implement automatic reconnection with exponential backoff.

**Approach:** Exponential backoff prevents hammering the access point when it's down or the network is congested. We start with a short retry delay (1 second) and double it on each consecutive failure, capping at a maximum (60 seconds). On successful reconnection, the delay resets. This is the same pattern used in cloud services (gRPC, AWS SDK retries, etc.) — the difference in embedded is that we're usually the only client, so the backoff is more about avoiding wasted CPU/radio cycles than protecting a shared server.

**Key concepts:**

- **`std::cmp::min` for cap** — Prevents the delay from growing unbounded.
- **`is_connected()` polling** — The main loop periodically checks connection status rather than relying on event callbacks. This is simpler and sufficient for most applications.
- **Reset on success** — The backoff state resets to 1s after a successful reconnect so the next failure recovers quickly.

```rust
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::{info, warn, error};
use std::time::Duration;

const WIFI_SSID: &str = "YourNetworkName";
const WIFI_PASS: &str = "YourPassword";

const INITIAL_BACKOFF_SECS: u64 = 1;
const MAX_BACKOFF_SECS: u64 = 60;

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 3 Exercise 3.1: WiFi with Exponential Backoff");

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

    // Initial connection (also uses backoff)
    connect_with_backoff(&mut wifi)?;

    let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
    info!("Connected! IP: {:?}", ip_info.ip);

    // Monitor loop — check connectivity and reconnect if dropped
    loop {
        std::thread::sleep(Duration::from_secs(5));

        if !wifi.is_connected()? {
            warn!("WiFi disconnected — starting reconnection...");
            connect_with_backoff(&mut wifi)?;
            let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
            info!("Reconnected! IP: {:?}", ip_info.ip);
        }
    }
}

fn connect_with_backoff(wifi: &mut BlockingWifi<EspWifi<'_>>) -> anyhow::Result<()> {
    let mut backoff_secs = INITIAL_BACKOFF_SECS;

    loop {
        info!("Attempting WiFi connection...");
        match wifi.connect() {
            Ok(_) => {
                wifi.wait_netif_up()?;
                info!("WiFi connected successfully");
                return Ok(());
            }
            Err(e) => {
                error!("Connection failed: {:?}", e);
                info!("Retrying in {}s...", backoff_secs);
                std::thread::sleep(Duration::from_secs(backoff_secs));
                backoff_secs = std::cmp::min(backoff_secs * 2, MAX_BACKOFF_SECS);
            }
        }
    }
}
```

---

## Solution 3.2: Store WiFi Credentials in NVS

**Exercise:** Store WiFi credentials in NVS (non-volatile storage) so they survive reboots.

**Approach:** NVS (Non-Volatile Storage) is the ESP-IDF equivalent of a persistent key-value store — think of it like an embedded `etcd` or Redis that survives power cycles. It's backed by a dedicated flash partition. We use it here to store SSID and password so the device can reconnect after reboot without hardcoded credentials.

The flow:

1. On boot, check NVS for stored credentials.
2. If found, use them to connect.
3. If not found (first boot), use hardcoded defaults and save them to NVS.

In a real product, you'd pair this with a provisioning mechanism (BLE, captive portal, or serial console) to set credentials initially. The hardcoded fallback here is just for the exercise.

**Key concepts:**

- **NVS namespaces** — Each `EspNvs::new` call takes a namespace string (max 15 chars). This scopes your keys to avoid collisions with other libraries or ESP-IDF components.
- **`get_str` / `set_str`** — NVS has typed accessors. Strings need a pre-allocated buffer for reads.
- **`EspDefaultNvsPartition`** — This is the same NVS partition the WiFi stack uses internally. It's safe to share; namespaces keep the data separate.

```rust
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::hal::prelude::*;
use esp_idf_svc::nvs::{EspDefaultNvsPartition, EspNvs, NvsDefault};
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use log::{info, warn};

// Fallback credentials for first boot
const DEFAULT_SSID: &str = "YourNetworkName";
const DEFAULT_PASS: &str = "YourPassword";

const NVS_NAMESPACE: &str = "wifi_creds"; // max 15 chars

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 3 Exercise 3.2: NVS WiFi Credentials");

    let peripherals = Peripherals::take()?;
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs_partition = EspDefaultNvsPartition::take()?;

    // Open NVS namespace for our wifi credentials
    let mut nvs = EspNvs::new(nvs_partition.clone(), NVS_NAMESPACE, true)?;

    // Try to load stored credentials, fall back to defaults
    let (ssid, password) = load_or_store_credentials(&mut nvs)?;
    info!("Using SSID: {}", ssid);

    // Connect with the loaded credentials
    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sys_loop.clone(), Some(nvs_partition))?,
        sys_loop,
    )?;

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: ssid.as_str().try_into().unwrap(),
        password: password.as_str().try_into().unwrap(),
        auth_method: AuthMethod::WPA2Personal,
        ..Default::default()
    }))?;

    wifi.start()?;
    wifi.connect()?;
    wifi.wait_netif_up()?;

    let ip_info = wifi.wifi().sta_netif().get_ip_info()?;
    info!("Connected! IP: {:?}", ip_info.ip);

    loop {
        std::thread::sleep(std::time::Duration::from_secs(10));
    }
}

fn load_or_store_credentials(nvs: &mut EspNvs<NvsDefault>) -> anyhow::Result<(String, String)> {
    let mut buf = [0u8; 64];

    let ssid = match nvs.get_str("ssid", &mut buf)? {
        Some(s) => {
            info!("Loaded SSID from NVS");
            s.to_string()
        }
        None => {
            warn!("No SSID in NVS — storing default");
            nvs.set_str("ssid", DEFAULT_SSID)?;
            DEFAULT_SSID.to_string()
        }
    };

    let password = match nvs.get_str("password", &mut buf)? {
        Some(s) => {
            info!("Loaded password from NVS");
            s.to_string()
        }
        None => {
            warn!("No password in NVS — storing default");
            nvs.set_str("password", DEFAULT_PASS)?;
            DEFAULT_PASS.to_string()
        }
    };

    Ok((ssid, password))
}
```

**Testing it:** Flash the firmware, then reboot the device. On the second boot, the serial log should show "Loaded SSID from NVS" instead of "No SSID in NVS — storing default". You can erase NVS to reset with `espflash erase-parts nvs` (or full flash erase with `espflash erase-flash`).
