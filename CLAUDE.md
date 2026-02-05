# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a documentation-only educational guide for embedded Rust development on ESP32 microcontrollers. It targets senior developers with Go/Linux/cloud backgrounds transitioning to embedded systems. There is no compiled source code, build system, or CI/CD in this repository — only markdown documentation with embedded Rust code examples.

**Target hardware:** ESP32-C6-DevKitC-1 (RISC-V) and ESP32-S3-WROOM-1 (Xtensa)
**Development environment:** ARM Mac (M1/M2/M3)
**License:** Apache 2.0

## Repository Structure

- `README.md` — Entry point with guide overview
- `docs/README.md` — Table of contents and guide structure
- `docs/part1/` — Environment setup (toolchain, ESP-IDF, USB drivers)
- `docs/part2/` — Rust fundamentals for Go developers (ownership, borrowing, error handling, traits, no_std)
- `docs/part3/` — ESP32 project structure (Cargo.toml, .cargo/config.toml, build.rs, sdkconfig.defaults)
- `docs/part4/` — Six progressive hands-on labs (LED blink → button → WiFi → HTTP server → ADC → MQTT)
- `docs/part5/` — Two final projects (Home Assistant sensor integration, Prometheus metrics exporter)
- `docs/part6/` — Troubleshooting and next steps
- `docs/references.md` — Comprehensive cheatsheet with 40+ code snippets, pin maps, Cargo.toml templates, sdkconfig reference

## Key Conventions

- All code examples use `esp-idf-svc`/`esp-idf-hal` (the ESP-IDF Rust bindings approach), not bare-metal `no_std`
- Labs build progressively — each lab assumes completion of prior labs
- Code examples are complete and runnable, not fragments
- The guide uses Go comparisons extensively (ownership vs GC, Result vs error tuples, traits vs interfaces)
- Two target architectures: `riscv32imac-esp-espidf` (C6) and `xtensa-esp32s3-espidf` (S3)

## When Editing This Guide

- Maintain consistency with the taught toolchain: `espup`, `espflash`, `cargo-generate`, `ldproxy`
- Key dependencies referenced throughout: `esp-idf-svc 0.49`, `esp-idf-hal 0.44`, `embedded-svc 0.28`, `anyhow 1.0`, `log 0.4`
- `docs/references.md` serves as the searchable companion to all parts — update it when adding new patterns or code snippets
- Hardware pin maps in references.md are specific to the DevKitC-1 and WROOM-1 boards
