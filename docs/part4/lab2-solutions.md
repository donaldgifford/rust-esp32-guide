# Lab 2 Solutions

<!--toc:start-->
- [Lab 2 Solutions](#lab-2-solutions)
  - [Solution 2.1: Long Press Detector](#solution-21-long-press-detector)
  - [Solution 2.2: Mode Toggle with Blink Speeds](#solution-22-mode-toggle-with-blink-speeds)
<!--toc:end-->

[Back to Part 4: Hands-On Labs](./README.md)

## Solution 2.1: Long Press Detector

**Exercise:** Implement a "long press" detector that triggers after holding for 2 seconds.

**Approach:** We track *when* a button press started using `esp_idf_svc::sys::esp_timer_get_time()`, which returns microseconds since boot. On each polling loop iteration, if the button is still held and the elapsed time exceeds 2 seconds, we trigger the long-press action. A `triggered` flag prevents the action from firing repeatedly while the button remains held.

**Why `esp_timer_get_time` instead of counting loop iterations?** Counting iterations (e.g. "200 loops × 10ms = 2 seconds") is fragile — FreeRtos::delay_ms isn't precise, and any work in the loop body adds drift. The hardware timer gives wall-clock accuracy regardless of loop timing.

**Key concepts:**

- **Edge detection** — We still detect the falling edge (press) and rising edge (release) from Lab 2's original code.
- **Timestamp-based duration** — More reliable than counting polling iterations for timing thresholds.
- **State machine** — The `triggered` flag adds a third state: not pressed, pressed-but-not-yet-long, and long-press-fired.

```rust
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use log::info;

const LONG_PRESS_US: i64 = 2_000_000; // 2 seconds in microseconds

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 2 Exercise 2.1: Long Press Detector");

    let peripherals = Peripherals::take()?;
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;
    let button = PinDriver::input(peripherals.pins.gpio9)?;

    let mut last_state = button.is_high();
    let mut press_start: i64 = 0;
    let mut triggered = false;

    loop {
        let current_state = button.is_high();
        let now = unsafe { esp_idf_svc::sys::esp_timer_get_time() };

        // Falling edge: button just pressed
        if last_state && !current_state {
            press_start = now;
            triggered = false;
            info!("Button pressed — timing...");
        }

        // Button is held down and we haven't triggered yet
        if !current_state && !triggered {
            let held_us = now - press_start;
            if held_us >= LONG_PRESS_US {
                triggered = true;
                led.set_high()?;
                info!("Long press detected! Held for {:.1}s", held_us as f64 / 1_000_000.0);
            }
        }

        // Rising edge: button released
        if !last_state && current_state {
            if !triggered {
                info!("Short press (released before 2s)");
            }
            led.set_low()?;
        }

        last_state = current_state;
        FreeRtos::delay_ms(10);
    }
}
```

---

## Solution 2.2: Mode Toggle with Blink Speeds

**Exercise:** Create a mode toggle — short press cycles through 3 LED blink speeds.

**Approach:** We decouple the button-reading logic from the LED-blinking logic using a simple state machine. A short press advances through three modes (slow/medium/fast), each with its own delay. The LED blinks continuously at the current speed, and button presses change the mode on the next cycle.

**Why separate the blink loop from button reads?** If we only check the button between full on/off cycles, a press during a long "slow" blink might be missed. Instead, we break the delay into small chunks and poll the button within each chunk. This keeps the UI responsive regardless of blink speed.

**Key concepts:**

- **Modular arithmetic for cycling** — `(mode + 1) % 3` wraps cleanly through modes 0, 1, 2.
- **Non-blocking delay pattern** — Splitting a long delay into small polling intervals so button presses are detected promptly.

```rust
use esp_idf_svc::hal::delay::FreeRtos;
use esp_idf_svc::hal::gpio::*;
use esp_idf_svc::hal::prelude::*;
use log::info;

const SPEEDS: [u32; 3] = [1000, 400, 100]; // slow, medium, fast (ms per half-cycle)
const SPEED_NAMES: [&str; 3] = ["Slow", "Medium", "Fast"];

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    info!("Lab 2 Exercise 2.2: Mode Toggle");

    let peripherals = Peripherals::take()?;
    let mut led = PinDriver::output(peripherals.pins.gpio8)?;
    let button = PinDriver::input(peripherals.pins.gpio9)?;

    let mut last_state = button.is_high();
    let mut mode: usize = 0;

    info!("Mode: {} ({}ms)", SPEED_NAMES[mode], SPEEDS[mode]);

    loop {
        // --- LED on phase ---
        led.set_high()?;
        if poll_button_during_delay(&button, &mut last_state, &mut mode, SPEEDS[mode]) {
            info!("Mode: {} ({}ms)", SPEED_NAMES[mode], SPEEDS[mode]);
        }

        // --- LED off phase ---
        led.set_low()?;
        if poll_button_during_delay(&button, &mut last_state, &mut mode, SPEEDS[mode]) {
            info!("Mode: {} ({}ms)", SPEED_NAMES[mode], SPEEDS[mode]);
        }
    }
}

/// Delay for `total_ms` in small increments, polling the button each iteration.
/// Returns true if the mode changed during this delay.
fn poll_button_during_delay(
    button: &PinDriver<'_, impl InputPin, Input>,
    last_state: &mut bool,
    mode: &mut usize,
    total_ms: u32,
) -> bool {
    let mut changed = false;
    let mut elapsed = 0u32;
    let poll_interval = 10u32;

    while elapsed < total_ms {
        let current = button.is_high();

        // Falling edge = button press
        if *last_state && !current {
            *mode = (*mode + 1) % SPEEDS.len();
            changed = true;
        }

        *last_state = current;
        FreeRtos::delay_ms(poll_interval);
        elapsed += poll_interval;
    }
    changed
}
```
