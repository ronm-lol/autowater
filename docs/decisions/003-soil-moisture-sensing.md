# ADR-003: Soil Moisture Sensing

## Status
Accepted

## Context
Each plant needs independent soil moisture monitoring to trigger watering only when soil is dry enough. Plants include succulents (trigger at very low moisture, ~20%) and tropicals (trigger at moderate moisture, ~40–60%). Smallest pots are 8cm diameter.

## Options Considered

| Option | Pros | Cons | Cost (13 sensors) |
|--------|------|------|------|
| **Capacitive analog sensor (generic V1.2)** | Cheap; no corrosion (vs. resistive); 16mm wide fits 8cm pots; analog output works with mux | ESP32 ADC non-linearity requires calibration; needs mux for Zone A | ~$15–30 CAD |
| Adafruit STEMMA Soil Sensor (I2C) | High accuracy; stable; clean HA integration | Fixed I2C address requires TCA9548A mux for multiple sensors; $7.50 USD each = ~$130 CAD for 13 | ~$130 CAD |
| Resistive soil sensor | Very cheap | Corrodes rapidly in soil; unreliable within months | Not viable |
| Chirp! I2C sensor | Accurate; I2C | Same multi-sensor address problem; ~$10 each | ~$130 CAD |

## Decision
**Generic capacitive soil moisture sensor V1.2 (analog output, 3.3V–5V).**

Dimensions: 99mm × 16mm. At 16mm wide, these fit comfortably in 8cm (80mm) diameter pots. Analog output is compatible with the CD4067 multiplexer strategy.

**Multiplexer for Zone A:** ESP32 ADC2 is unusable when WiFi is active; ADC1 has 8 channels (GPIO32–39). Zone A needs 10 sensors. Solution: CD4067 16-channel analog multiplexer driven by 4 ESP32 GPIO select pins, routing one sensor at a time to a single ADC1 pin. Zone B (2–3 sensors) connects directly to ADC1 without a mux.

**Calibration requirement:** ESP32 ADC is non-linear. Every sensor must be individually calibrated using ESPHome's `calibrate_linear` filter:
- Dry calibration: hold sensor in air → record raw ADC value (≈ 3100–3300)
- Wet calibration: submerge sensor tip in water → record raw ADC value (≈ 1200–1500)
- Map to 0–100% moisture range

Per-plant moisture thresholds (the % at which watering is triggered) are set in the ESPHome YAML config and can be updated via OTA.

## Consequences
- Enables: per-plant moisture-triggered watering; configurable thresholds per plant type
- Prevents: simultaneous multi-sensor reading (not needed for this application)
- Risks: generic sensors have unit-to-unit variance — calibration is mandatory, not optional; sensor accuracy degrades in very dry or very dense soil; sensors are not waterproof above the PCB line

## Kill Switch
If generic sensors prove unreliable after 6 months (readings drift or become uncorrelated with actual moisture), replace with Adafruit STEMMA sensors + TCA9548A I2C mux. The ESPHome config changes but GPIO wiring is minimal.
