# ADR-001: Microcontroller and Firmware Platform

## Status
Accepted

## Context
Each zone unit needs a microcontroller that can:
- Read 2–10 analog soil moisture sensors
- Control 3–11 relay channels (solenoid valves + pump)
- Monitor 2–3 digital flood sensor inputs with interrupt support
- Run all watering decision logic locally (offline-safe)
- Integrate with Home Assistant without a custom server
- Run on 3.3V or 5V from a 12V supply

## Options Considered

| Option | Pros | Cons | Cost |
|--------|------|------|------|
| **ESP32 + ESPHome** | Native HA integration; offline-first; rich sensor/actuator library; active community; OTA updates | ADC2 unusable with WiFi active (only 8 ADC1 channels); ADC noisy/non-linear | ~$8–15 CAD/board |
| ESP8266 + ESPHome | Same ESPHome ecosystem; cheaper | Only 1 ADC pin; insufficient GPIO for Zone A without significant expansion | ~$5–8 CAD |
| Raspberry Pi Zero 2W + Python | Full Linux; easy to debug; any library | Overkill; slower boot; SD card reliability; more power draw; no native ESPHome | ~$20–30 CAD |
| Arduino Mega + custom firmware | Plenty of GPIO and ADC; simple | No WiFi without shield; no HA integration without extra work; no OTA | ~$15 CAD |

## Decision
**ESP32 + ESPHome.**

ESPHome is the dominant platform for DIY HA-connected sensor nodes. It generates firmware from a YAML config — no custom C++ needed. The native HA integration is zero-config (autodiscovery). All logic (moisture thresholds, watering sequences, flood response) runs on-device. OTA updates are supported. The ADC limitations are worked around with a CD4067 multiplexer (see ADR-003).

Specific board: **ESP32-WROOM-32 DevKit v1** (30-pin or 38-pin). Widely available, breadboard-compatible, well-documented pinout. Avoid Adafruit HUZZAH32 for this build — the LiPo connector and form factor add cost without benefit since we're mains-powered.

## Consequences
- Enables: HA integration, OTA firmware updates, offline operation, per-plant automation logic in YAML
- Prevents: easy use of ADC2 pins (WiFi conflict) — multiplexer required for Zone A
- Risks: ESP32 ADC non-linearity requires per-sensor calibration; ESPHome YAML has limits on complex logic (may need lambda/C++ for advanced sequencing)

## Kill Switch
If ESPHome's automation model proves too limiting for the sequencing logic, or if WiFi reliability in the installation location is poor (>5% packet loss to HA), migrate Zone A to Arduino Mega + ESP01 WiFi module, or add an external I2C ADC expander and use the native ESP32 Arduino SDK directly.
