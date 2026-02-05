# Lab 1 Solutions

<!--toc:start-->
- [Lab 1 Solutions](#lab-1-solutions)
  - [Solution 1.1: SOS Morse Code Blink](#solution-11-sos-morse-code-blink)
  - [Solution 1.2: Blink Counter with Logging](#solution-12-blink-counter-with-logging)
<!--toc:end-->

[Back to Part 4: Hands-On Labs](./README.md)

## Solution 1.1: SOS Morse Code Blink

**Exercise:** Modify the blink pattern to blink SOS in Morse code (... --- ...).

**Approach:** Rather than hardcoding a sequence of `set_high`/`set_low`/`delay` calls, we define Morse timing constants and represent the entire SOS pattern as data — a slice of `(bool, u32)` tuples. A helper function iterates the slice and drives the LED. This makes it straightforward to add other Morse patterns later without duplicating control-flow logic.

**Key concepts:**

- **Generic pin types** — `PinDriver<'_, impl OutputPin, Output>` lets `blink_pattern` work with any GPIO pin configured as output, not just a specific pin number.
- **Morse timing** — A dash is 3x the length of a dot, gaps between elements equal one dot length, and gaps between letters equal one dash length.
- **Data-driven patterns** — Encoding the pattern as a `Vec<(bool, u32)>` separates the "what to blink" from the "how to blink" logic.

```rust
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use log::info;

/// Drive an LED through a sequence of (on/off, duration_ms) steps.
fn blink_pattern(
    led: &mut PinDriver<'_, impl OutputPin, Output>,
    pattern: &[(bool, u32)],
) -> anyhow::Result<()> {
    for (state, duration_ms) in pattern {
        if *state {
            led.set_high()?;
        } else {
            led.set_low()?;
        }
        FreeRtos::delay_ms(*duration_ms);
    }
    Ok(())
}

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 1 Exercise 1.1: SOS Morse Code");

    let peripherals = Peripherals::take()?;
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;

    // Morse timing constants (milliseconds)
    const DOT: u32 = 150;
    const DASH: u32 = 450;      // 3x dot
    const GAP: u32 = 150;       // gap between elements within a letter
    const LETTER_GAP: u32 = 450; // gap between letters (3x dot)
    const WORD_GAP: u32 = 1050;  // gap between repetitions (7x dot)

    // SOS pattern: ... --- ...
    let sos: Vec<(bool, u32)> = vec![
        // S: dot dot dot
        (true, DOT), (false, GAP),
        (true, DOT), (false, GAP),
        (true, DOT), (false, LETTER_GAP),
        // O: dash dash dash
        (true, DASH), (false, GAP),
        (true, DASH), (false, GAP),
        (true, DASH), (false, LETTER_GAP),
        // S: dot dot dot
        (true, DOT), (false, GAP),
        (true, DOT), (false, GAP),
        (true, DOT), (false, WORD_GAP),
    ];

    loop {
        info!("SOS");
        blink_pattern(&mut led, &sos)?;
    }
}
```

---

## Solution 1.2: Blink Counter with Logging

**Exercise:** Add a variable `blink_count` and log how many times the LED has toggled.

**Approach:** This is intentionally simple — a `u32` counter incremented on each toggle. The interesting embedded consideration is what happens when `blink_count` overflows. On a standard Rust build, `u32` arithmetic wraps in release mode and panics in debug mode. We use `wrapping_add` to be explicit about the intent. On an ESP32 blinking at 1 Hz, a `u32` overflows after ~136 years, so this is academic — but being explicit about overflow behavior is a good habit in embedded code where panics are expensive.

```rust
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use log::info;

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 1 Exercise 1.2: Blink Counter");

    let peripherals = Peripherals::take()?;
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;

    let mut blink_count: u32 = 0;

    loop {
        led.set_high()?;
        blink_count = blink_count.wrapping_add(1);
        info!("LED ON  — toggle count: {}", blink_count);
        FreeRtos::delay_ms(500);

        led.set_low()?;
        blink_count = blink_count.wrapping_add(1);
        info!("LED OFF — toggle count: {}", blink_count);
        FreeRtos::delay_ms(500);
    }
}
```
