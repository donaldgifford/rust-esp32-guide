# Part 2: Rust Fundamentals for Go Developers

<!--toc:start-->
- [Part 2: Rust Fundamentals for Go Developers](#part-2-rust-fundamentals-for-go-developers)
  - [2.1 Ownership and Borrowing](#21-ownership-and-borrowing)
  - [2.2 Error Handling: Result vs Go's (value, error)](#22-error-handling-result-vs-gos-value-error)
  - [2.3 Traits vs Go Interfaces](#23-traits-vs-go-interfaces)
  - [2.4 no_std: Rust Without the Standard Library](#24-nostd-rust-without-the-standard-library)
  - [2.5 Key Syntax Quick Reference](#25-key-syntax-quick-reference)
<!--toc:end-->

This section covers Rust concepts that differ most from Go. We'll focus on what matters for embedded development.

## 2.1 Ownership and Borrowing

In Go, the garbage collector manages memory. In Rust, ownership rules enforce memory safety at compile time with zero runtime cost—critical for microcontrollers with limited resources.

**The Three Rules:**

1. Each value has exactly one owner
2. When the owner goes out of scope, the value is dropped
3. You can have either one mutable reference OR any number of immutable references

```rust
// Go equivalent thinking
// In Go: data := []byte{1, 2, 3}  // GC tracks this
// In Go: process(data)            // data copied or referenced, GC handles it

// Rust: Ownership transfer (move)
fn main() {
    let data = vec![1, 2, 3];  // data owns the vector
    process(data);              // ownership MOVED to process
    // println!("{:?}", data);  // ERROR: data no longer valid here
}

fn process(d: Vec<u8>) {
    println!("{:?}", d);
}  // d is dropped here, memory freed

// Rust: Borrowing (like passing a pointer, but safe)
fn main() {
    let data = vec![1, 2, 3];
    process(&data);             // lend immutable reference
    println!("{:?}", data);     // OK: we still own data
}

fn process(d: &Vec<u8>) {       // borrows, doesn't own
    println!("{:?}", d);
}

// Rust: Mutable borrowing
fn main() {
    let mut data = vec![1, 2, 3];
    modify(&mut data);          // lend mutable reference
    println!("{:?}", data);     // [1, 2, 3, 4]
}

fn modify(d: &mut Vec<u8>) {
    d.push(4);
}
```

**Why this matters for embedded:** No garbage collector means predictable timing and minimal RAM usage. A 320KB RAM ESP32-C6 can't afford GC pauses.

## 2.2 Error Handling: Result vs Go's (value, error)

Go's pattern:

```go
data, err := readSensor()
if err != nil {
    return err
}
```

Rust's pattern:

```rust
// Result<T, E> is an enum: either Ok(T) or Err(E)

fn read_sensor() -> Result<u32, SensorError> {
    // ... implementation
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Pattern 1: Match (like a switch on the result)
    match read_sensor() {
        Ok(value) => println!("Temperature: {}", value),
        Err(e) => println!("Error: {:?}", e),
    }

    // Pattern 2: The ? operator (like Go's if err != nil { return err })
    let value = read_sensor()?;  // returns early if Err
    println!("Temperature: {}", value);

    // Pattern 3: unwrap (panics on error - use sparingly!)
    let value = read_sensor().unwrap();  // PANIC if Err

    // Pattern 4: unwrap_or (default value)
    let value = read_sensor().unwrap_or(0);

    Ok(())
}
```

**Embedded consideration:** `panic!` on a microcontroller typically causes a reset. Use `?` propagation or explicit error handling.

## 2.3 Traits vs Go Interfaces

Go interfaces are implicit—any type with the right methods implements the interface. Rust traits are explicit.

```rust
// Define a trait (like a Go interface)
trait Sensor {
    fn read(&self) -> Result<f32, SensorError>;
    fn name(&self) -> &str;
}

// Implement the trait for a type
struct TemperatureSensor {
    pin: u8,
}

impl Sensor for TemperatureSensor {
    fn read(&self) -> Result<f32, SensorError> {
        // Read from hardware...
        Ok(25.5)
    }

    fn name(&self) -> &str {
        "Temperature"
    }
}

// Use the trait as a bound (like Go's interface parameter)
fn log_reading<S: Sensor>(sensor: &S) {
    match sensor.read() {
        Ok(v) => println!("{}: {}", sensor.name(), v),
        Err(e) => println!("Error reading {}: {:?}", sensor.name(), e),
    }
}

// Or use dynamic dispatch (like Go's interface{})
fn log_reading_dyn(sensor: &dyn Sensor) {
    // Same implementation, but uses vtable like Go interfaces
}
```

## 2.4 no_std: Rust Without the Standard Library

Embedded Rust often runs without an operating system, meaning no heap allocator, no filesystem, no threads. The `#![no_std]` attribute tells Rust to use `core` (always available) instead of `std`.

```rust
#![no_std]  // No standard library

// Available in no_std:
use core::fmt::Write;           // Formatting
use core::result::Result;       // Error handling
use core::option::Option;       // Optional values

// NOT available without extra work:
// - String, Vec (need an allocator)
// - println! (need stdout)
// - std::thread (need an OS)
// - std::net (need a network stack)
```

**Good news:** ESP-IDF provides an allocator and FreeRTOS, so we can use most of `std` with `esp-idf-hal`. We're doing "std on embedded."

## 2.5 Key Syntax Quick Reference

```rust
// Variables (immutable by default, opposite of Go)
let x = 5;              // immutable
let mut y = 5;          // mutable

// Type annotations
let x: i32 = 5;         // explicit type
let x = 5i32;           // suffix notation

// Functions
fn add(a: i32, b: i32) -> i32 {
    a + b               // no semicolon = return value (expression)
}

// Structs
struct Config {
    ssid: String,
    password: String,
    port: u16,
}

// Instantiation
let cfg = Config {
    ssid: String::from("MyWiFi"),
    password: String::from("secret"),
    port: 8080,
};

// Impl blocks (methods)
impl Config {
    // Associated function (like Go's NewConfig)
    fn new(ssid: &str, password: &str) -> Self {
        Self {
            ssid: ssid.to_string(),
            password: password.to_string(),
            port: 8080,
        }
    }

    // Method (has &self)
    fn connection_string(&self) -> String {
        format!("{}:{}", self.ssid, self.port)
    }
}

// Enums (much more powerful than Go's const iota)
enum SensorReading {
    Temperature(f32),
    Humidity(f32),
    Pressure { value: f32, unit: String },
    Error(String),
}

// Pattern matching
match reading {
    SensorReading::Temperature(t) => println!("Temp: {}", t),
    SensorReading::Humidity(h) if h > 80.0 => println!("High humidity!"),
    SensorReading::Humidity(h) => println!("Humidity: {}", h),
    SensorReading::Pressure { value, .. } => println!("Pressure: {}", value),
    SensorReading::Error(msg) => eprintln!("Error: {}", msg),
}

// Closures (like Go's anonymous functions)
let add = |a, b| a + b;
let double = |x| x * 2;
let result = add(1, 2);

// Iterators (more functional than Go's for loops)
let numbers = vec![1, 2, 3, 4, 5];
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
let sum: i32 = numbers.iter().sum();
let evens: Vec<&i32> = numbers.iter().filter(|x| *x % 2 == 0).collect();
```
