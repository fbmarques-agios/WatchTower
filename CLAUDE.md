# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working in this repository.

## Project Overview

WatchTower is a DIY fake WWVB transmitter: an ESP32 generates a 60 kHz carrier on a GPIO and pulse-width modulates it with the WWVB time code, so radio-controlled watches (Casio multiband, Citizen multiband) sync to NTP-derived time without a real WWVB signal from Fort Collins.

This fork (`fbmarques-agios/WatchTower`) adapts the project for the **M5StickC Plus 1.1** (ESP32-PICO-D4, 4 MB flash). Upstream (`emmby/WatchTower`) targets the Adafruit QT Py ESP32 + DRV8833 H-bridge and is preserved as a parallel build env.

What this fork adds on top of upstream:
- `m5stickc_plus` PlatformIO env (board variant override, partition table for 4 MB flash)
- LCD support via M5Unified (header, boot status, live date/time/IP)
- `ANTENNA_DRIVE_LEVEL` constant (0-3) tuning the GPIO output drive strength
- Brazil timezone default (`BRT3`, no DST)
- PT-BR fork-specific README (replaces the upstream English one)

## Architecture

```
NTP (pool.ntp.org)
  ─▶ configTzTime          (ESP32 keeps wall-clock)
  ─▶ wwvbLogicSignal()     (encodes the current second's bit, 60-bit frame)
  ─▶ ledcWrite(PIN_ANTENNA, 50% or 0%)   (60 kHz LEDC PWM, 8-bit res)
  ─▶ GPIO26                (M5Stick) / GPIO13 (QT Py)
  ─▶ coil  [+ optional H-bridge amplifier + ferrite rod]
  ─▶ watch's ferrite antenna picks up the magnetic field
```

WiFi: WiFiManager captive portal `WatchTower` on first boot. Status: Web UI `http://watchtower.local` via ESPUI; serial 115200; LCD via M5Unified (M5Stick env only).

## Build environments (`platformio.ini`)

| Env | Board / variant | Purpose |
|---|---|---|
| `m5stickc_plus` (default) | `m5stick-c` + `board_build.variant = m5stack_stickc_plus`, partition `huge_app.csv` | This fork's target |
| `adafruit_qtpy_esp32` | `adafruit_qtpy_esp32`, partition `default_8MB.csv` | Upstream's original target |
| `native` | host | Unity tests (`test/test_native/`) — compiles `WatchTower.ino` with `-D UNIT_TEST` |

The two firmware envs share `[esp32_base]` (platform, framework, lib_deps). `m5stickc_plus` extends with `m5stack/M5Unified` so the QT Py env is unaffected.

## Key files

| Path | Purpose |
|---|---|
| `WatchTower.ino` | Firmware: setup, loop, WWVB encoder, LCD display |
| `platformio.ini` | Build envs and dependencies |
| `customJS.h` | Custom JS injected into the ESPUI page |
| `test/test_native/test_bootstrap.cpp` | Unity tests for WWVB encoding (includes `WatchTower.ino` directly) |
| `test/mocks/*.h` | Arduino / ESP / ESPUI / WiFi mocks used by the native test build |
| `README.md` | PT-BR user manual for this fork (M5StickC Plus path + Citizen H874 specifics) |
| `enclosure/The Watch Tower.{stl,f3d}` | 3D-printable case (upstream — fits QT Py, NOT M5Stick) |

## Commands

```bash
# PlatformIO Core lives in its own venv (created by get-platformio.py)
PIO=~/.platformio/penv/bin/pio

# Build firmware (default env = m5stickc_plus)
$PIO run

# Upload + monitor (115200 baud)
$PIO run -e m5stickc_plus -t upload --upload-port /dev/ttyUSB0
$PIO device monitor -e m5stickc_plus

# Native unit tests (host)
$PIO test -e native

# Git hooks (pre-commit: build, pre-push: build + native tests)
git config core.hooksPath .githooks
```

**Serial port permission:** the M5StickC Plus enumerates as `/dev/ttyUSB0` with mode `crw-rw---- root dialout`. Either `sudo chmod 666 /dev/ttyUSB0` (temporary, resets on replug) or `sudo usermod -aG dialout $USER` (permanent, requires re-login).

## Design decisions & dev-time gotchas

### M5StickC Plus specifics
- **`PIN_ANTENNA = GPIO26`** — GPIO13 (upstream default) drives the M5Stick's LCD ST7789V2.
- **Variant override** — `board = m5stick-c` in PlatformIO points at variant `m5stick_c`, which is missing from current arduino-esp32 cores. Force `board_build.variant = m5stack_stickc_plus`.
- **Partition table** — `huge_app.csv` (3 MB single app, no OTA). The default 4 MB partition doesn't fit ESPUI + AsyncWebServer + WiFiManager + M5Unified.
- **No NeoPixel** — `pixel = NULL` (the `#ifdef PIN_NEOPIXEL` falls through). All pixel calls are already guarded with `if(pixel)`, so nothing else changes.
- **M5Unified scoping** — only the `m5stickc_plus` env adds `m5stack/M5Unified` to `lib_deps`. The QT Py env is unchanged.

### Antenna drive strength
- `ANTENNA_DRIVE_LEVEL` (0-3) maps to `GPIO_DRIVE_CAP_0..3` via `gpio_set_drive_capability` called after `ledcAttach`.
- Approx currents: 0 → ~5 mA, 1 → ~10 mA, 2 → ~20 mA (ESP32 default), 3 → ~40 mA.
- Level 3 without a series resistor stresses the pin on **low-impedance coils** (air-core hookup wire). Safe with **high-inductance coils** (ferrite rod with many turns — reactance ~100+ Ω at 60 kHz limits current naturally).
- The amplifier path in the "definitive build" makes drive level irrelevant — the GPIO drives a high-impedance amplifier input.

### WWVB encoding
- Frame = 60 seconds, one bit per second. Bit types: ZERO (low 200 ms / high 800 ms), ONE (500/500), MARK (800/200).
- Transmitted: hour, minute (UTC), day-of-year, year, leap-year flag, DST flags (today + tomorrow), markers.
- UT1 correction and leap-second bits are hard-coded zero — irrelevant for normal watch sync.
- `timezone = "BRT3"` (UTC-3, no DST). The Citizen H874 maps US timezone offsets (-3 .. -8) to the WWVB station; UTC-3 = Brasília time AND a WWVB-listening setting.

### Build / tooling
- Platform: `pioarduino/platform-espressif32` (community fork required for arduino-esp32 3.x and the `ledcAttach` API).
- Native test build (`-D UNIT_TEST`) `#include`s `WatchTower.ino` directly via `test_bootstrap.cpp`. M5Unified / `driver/gpio.h` / IPAddress::toString / `constrain` are unavailable there — guarded with `#ifndef UNIT_TEST` and replaced with no-op stubs.
- Pre-commit hook: full `pio run -e m5stickc_plus` (cached ~10-20 s) — catches syntax errors.
- Pre-push hook: same build + `pio test -e native`.

### Git / publishing
- Two remote scenarios: SSH (`git@github.com:fbmarques-agios/...`) like sibling project `soc-bot-abasp`.
- `p.pdf` (Citizen H874 user manual) is in `.gitignore` — copyright Citizen, not redistributable.
- `.pio/` (build artifacts) is in upstream's `.gitignore`. `.claude/` is global-ignored by Claude Code.

## Documentation routing

| Doc | Content | When to update |
|---|---|---|
| `CLAUDE.md` (this file) | Architecture, build envs, dev-time landmines, key decisions — max ~150 lines | New env, new firmware decision, new dev-time landmine |
| `README.md` | PT-BR end-user manual for this fork: install, flashing, coil wiring, watch setup, troubleshooting (H874-specific). Replaces the upstream English README | New hardware step, new user-facing behavior, status updates |
| `enclosure/` | Upstream 3D model (QT Py form factor) | Don't edit — upstream owns it. M5Stick-specific enclosure will live under a new path |

End-user instructions (where to plug what, how to use the watch) → `README.md`. Firmware-side decisions, build-system quirks, hidden invariants → here.
