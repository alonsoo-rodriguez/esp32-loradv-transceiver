ESP32 LoRa DV Transceiver
=========================

ESP32 LoRaDV is a digital voice (DV) handheld transceiver platform built on the ESP32.  
It combines a LoRa/FSK RF front‑end with low‑bitrate voice codecs (Codec2 and Opus), an OLED UI, rotary encoder, and power‑managed firmware, targeting 433/446 MHz unlicensed bands.

This repository contains the firmware, hardware schematics, and mechanical CAD for a fully integrated handheld, excluding the battery cell and charge controller.

---

## Features

- **ESP32‑based DV handheld**: Uses a standard ESP32 dev board as the main MCU.
- **LoRa and FSK modulation**:
  - LoRa with configurable frequency, bandwidth, spreading factor, coding rate, power, preamble, and CRC.
  - FSK with configurable bitrate, deviation, and RX bandwidth.
- **Digital voice**:
  - Codec2 support with configurable mode and packet aggregation.
  - Optional Opus codec with configurable bitrate and frame size.
- **Audio I/O**:
  - I2S microphone and speaker interfaces with configurable sample rate and resampling.
  - High‑pass filtering and volume control, with persistent volume state.
- **User interface**:
  - 128×32 OLED (SSD1306) display.
  - Rotary encoder with push button for navigation and settings.
  - Dedicated PTT button.
- **Power management & monitoring**:
  - Light‑sleep scheduling and configurable duty‑cycle.
  - Battery voltage monitoring and calibration.
  - Hardware monitoring hooks.
- **Persistent configuration**:
  - `Preferences`‑backed configuration with versioning and reset.
  - Menu‑driven settings with structured configuration model.
- **Hardware assets**:
  - KiCad schematics, PCB, and Gerber files for the RF/logic board.
  - 3D CAD models for the handheld enclosure and antenna parts.

---

## Repository Structure

- **`src/`** – Firmware implementation:
  - `main.cpp`: Arduino entry point; initializes `Config` and `Service`.
  - `service.cpp`: High‑level orchestration of radio, audio, UI, power, and monitoring.
  - `audio/`: Audio codecs (Codec2, Opus) and audio task logic.
  - `hal/`: Hardware‑abstraction tasks (radio, power management, hardware monitor).
  - `settings/`: Configuration loading, defaults, and settings menu implementation.
  - `utils/`: Utility and DSP helpers.
- **`include/`** – Public headers for the above modules (audio, HAL, settings, utils, service).
- **`variants/`** – Per‑hardware build variants with `platformio.ini` fragments and pin mappings:
  - `esp32dev_ra01/`: ESP32 + Ra‑01 style LoRa module.
  - `esp32dev_e22/`: ESP32 + E22 LoRa module (SX127x‑based).
  - `esp32dev_e22_sx1262/`: ESP32 + E22 module based on SX1262.
- **`extras/schematics/`** – KiCad project for the RF/logic board:
  - `*.kicad_sch`, `*.kicad_pcb`, symbol/footprint libraries and netlists.
  - `gerber/`: Fabrication output for PCB manufacturing.
  - `README.md`: Short overview and board images.
- **`extras/cad/`** – Mechanical CAD for the handheld enclosure:
  - Case base, upper shell, antenna base/cover, encoder knob, PTT button, battery lid, etc.
  - `README.md`: Details on the antenna mechanical tuning and parts.

---

## Hardware Overview

At a high level the system is composed of:

- **MCU**: ESP32 devkit (30‑pin wide variants supported by the provided footprints).
- **RF front‑end**:
  - LoRa modules such as Ra‑01, E22 (SX127x), or E22‑400M30S (SX1262), selected via firmware variant.
  - Configurable RX/TX frequency range enforced by per‑variant limits.
- **User I/O**:
  - SSD1306 OLED display on I²C.
  - Rotary encoder (A/B channels, push button, optional VCC pin).
  - PTT button wired to a dedicated GPIO.
- **Audio path**:
  - I2S microphone and speaker (e.g., MAX98357A‑based amplifier).
  - Audio sample rate and codec sample rate conversions handled in firmware.
- **Power**:
  - External 18650 cell and boost/charge circuitry (not on the main board).
  - Battery voltage sensing via ADC for runtime monitoring and UI display.

Consult `extras/schematics/README.md` and `extras/cad/README.md` for board and enclosure details, including antenna construction and enclosure parts.

---

## Firmware Architecture

The core of the firmware is the `Service` class (`service.h` / `service.cpp`), which wires together:

- **`Config`** (`settings/config.h`/`.cpp`):
  - Holds all runtime configuration: modulation, RF, audio, pinouts, power management, privacy, and UI parameters.
  - Loads/saves from NVS via `Preferences`, with support for default initialization and reset.
- **Radio stack** (`hal/radio_task.h`/`.cpp`):
  - Uses RadioLib to configure and operate the LoRa/FSK transceiver.
  - Applies variant‑specific pin configuration and frequency limits.
- **Audio stack** (`audio/audio_task.h`/`.cpp`, `audio_codec_*`):
  - Manages I2S audio capture/rendering and codec integration.
  - Implements Codec2 and optionally Opus processing, framing, and packetization.
- **Power management** (`hal/pm_service.h`/`.cpp`):
  - Controls light‑sleep scheduling and wakeup/active timings based on configuration.
- **Hardware monitoring** (`hal/hw_monitor.h`/`.cpp`):
  - Tracks system measurements such as battery and potentially temperature/current.
- **Settings menu / UI** (`settings/settings_menu.h`/`.cpp`):
  - Provides a menu‑driven UI using the rotary encoder, encoder button, and display.

The Arduino `setup()` and `loop()` functions in `src/main.cpp` simply bootstrap the configuration and delegate to `Service`:

- Initialize serial output for debugging.
- Construct `Config`, load persisted settings (or defaults).
- Construct `Service` with the configuration and call `setup()`.
- In `loop()`, call `Service::loop()` and introduce a small delay (10 ms) to limit CPU load.

---

## Building the Firmware

This project targets **PlatformIO** with the **Arduino** framework for ESP32.

Because the repository contains **variant‑local `platformio.ini` files** under `variants/`, you have two main options for building:

### 1. Use a Variant `platformio.ini` as Your Project Root

1. In your IDE (e.g., VS Code with PlatformIO or CLion with PlatformIO), **open the folder** of the desired variant as the PlatformIO project:
   - `variants/esp32dev_ra01/`
   - `variants/esp32dev_e22/`
   - `variants/esp32dev_e22_sx1262/`
2. Ensure the source directory is set to the repository root (or adjust `platformio.ini` to point to `../../src` and `../../include` if necessary).
3. Select the appropriate environment, for example:
   - `[env:esp32dev_ra01]`
   - `[env:esp32dev_e22]` or `[env:esp32dev_e22_pm_optimize]`
   - `[env:esp32dev_e22_sx1262]` or `[env:esp32dev_e22_sx1262_pm_optimize]`
4. Run **Build** and **Upload** via PlatformIO.

The `*_pm_optimize` environments enable additional power‑management optimizations (longer sleep duration, reduced awake time) via preprocessor defines.

### 2. Create a Top‑Level `platformio.ini` (Recommended for New Users)

If you prefer a single top‑level entry point:

1. Create a `platformio.ini` at the repository root.
2. Define environments that match the variant configurations (include paths, defines, board type) and set:
   - `src_dir = src`
   - `include_dir = include`
3. Reuse or `#include` the environment sections from the existing variant files as appropriate.

This approach lets you build all variants from a single PlatformIO project.

---

## Configuration and Runtime Settings

Configuration is managed by the `Config` class:

- **Storage**: Persisted via ESP32 `Preferences` under a dedicated namespace.
- **Defaults**: Populated from `settings/default_config.h` in `Config::InitializeDefault()`.
- **Runtime access**: Passed as a shared pointer into `Service`, radio, audio, and other modules.

Key configuration areas:

- **Modulation**:
  - `ModType`: LoRa vs FSK mode.
  - LoRa: RX/TX frequencies, step, bandwidth, spreading factor, coding rate, power, sync word, CRC mode, preamble length.
  - FSK: bit rate, deviation, RX bandwidth, shaping.
- **Audio**:
  - Codec selection (Codec2 vs Opus), codec sample rate, I2S sample rate, resampling coefficient.
  - High‑pass filter cutoff, volume limits, current volume.
- **Privacy**:
  - Enable/disable flag and 32‑byte key for basic privacy.
- **Hardware mappings**:
  - LoRa pins (SS, RST, DIO/BUSY, RX/TX enable).
  - Encoder pins and step count.
  - I2S microphone/speaker pins.
  - Battery monitor ADC pin and calibration.
- **Power management**:
  - Sleep‑after timeout.
  - Light‑sleep duration and active window.
- **PTT**:
  - Dedicated GPIO pin for PTT button input.

Settings can typically be modified via the on‑device settings menu; you can also extend the menu and defaults in the `settings/` module.

---

## Hardware and Mechanical Assets

- **Schematics and PCB**:
  - See `extras/schematics/` for the full KiCad project, Gerbers, and symbol/footprint libraries.
  - The KiCad project targets the LoRaDV transceiver board with ESP32 devkit footprints and LoRa modules.
- **Enclosure and Antenna**:
  - See `extras/cad/` for case parts and antenna fixtures.
  - The bundled README describes antenna tuning using a helical 1.3 mm copper wire on an 8 mm former, trimmed to match the target band (433/446 MHz) with the cover installed.

These assets are sufficient to fabricate the PCB and 3D‑print the enclosure for a complete handheld build.

---

## Extending and Customizing

As a starting point for experimentation you can:

- **Add new modulation profiles** by extending the configuration model and radio initialization logic.
- **Integrate additional codecs** by following the patterns in `audio_codec_codec2.*` and `audio_codec_opus.*`.
- **Customize the UI** by extending `SettingsMenu` and the OLED rendering in `Service`.
- **Refine power behavior** by adjusting the PM configuration and `PmService` scheduling logic.

When making changes that affect persistent configuration, remember to bump the configuration version and handle migration or reset appropriately.

---

## License

See the license file(s) in `extras/schematics` and the root of this repository for detailed licensing information covering firmware, hardware, and mechanical assets.

