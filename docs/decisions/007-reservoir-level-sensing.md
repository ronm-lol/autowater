# ADR-007: Reservoir Level Sensing

## Status
Accepted

## Context
Each zone has a manually-filled reservoir. When it runs dry, the pump runs without water (risk of pump damage) and plants go unwatered. The user wants a continuous water level reading exposed in Home Assistant (e.g., "Zone A reservoir at 34%"), not just a low-water binary alert.

## Options Considered

| Option | Pros | Cons | Cost |
|--------|------|------|------|
| **VL53L0X laser ToF sensor (I2C)** | Native ESPHome support; 3cm–120cm range; I2C (no extra GPIO); 3.3V native; compact breakout | Not waterproof (must mount above water, not in it) | ~$4–8 CAD |
| JSN-SR04T waterproof ultrasonic | Waterproof probe | 20–25cm minimum range — cannot read near-full levels in a small reservoir; 2 GPIO pins | ~$8–12 CAD |
| VL53L1X laser ToF (I2C) | Longer range (4m) | Community-only ESPHome support (not native); longer range not needed here | ~$5–10 CAD |
| Float switch | Simple; cheap | Binary only (low / not-low); no continuous level reading | ~$3–5 CAD |

## Decision
**VL53L0X laser ToF sensor, mounted above the open reservoir pointing down.**

The VL53L0X has a 3cm minimum range — works reliably even when the reservoir is nearly full. I2C interface shares the bus with no GPIO cost (pins GPIO21/GPIO22 on ESP32). Native ESPHome `vl53l0x` component requires no custom code.

The sensor is not waterproof, but it doesn't need to be: it mounts above the reservoir opening on a small 3D-printed bracket, pointing straight down at the water surface. Laser light reflects off the water surface cleanly.

### Mounting

A small bracket (designed as part of the enclosure 3D print order — see ADR-006) clips to the reservoir rim or sits on top of it. The sensor faces down; cable routes to the electronics enclosure. For minimal/modern aesthetic: the bracket is same PETG material/colour as the main enclosure.

### ESPHome Configuration (concept)

```yaml
i2c:
  sda: GPIO21
  scl: GPIO22

sensor:
  - platform: vl53l0x
    name: "Zone A Reservoir Level"
    update_interval: 60s
    filters:
      - calibrate_linear:
          # Calibrate after physical installation:
          # measure sensor-to-surface distance at full and empty
          - 0.05 -> 100   # 5cm distance = full (adjust to actual)
          - 0.30 -> 0     # 30cm distance = empty (adjust to actual)
      - clamp:
          min_value: 0
          max_value: 100
    unit_of_measurement: "%"
```

Calibration is done once after installation by measuring the actual sensor-to-water-surface distances at full and empty.

### Home Assistant Behaviour

- HA entity: `sensor.zone_a_reservoir_level` (0–100%)
- HA automation: alert when level drops below 20% (threshold configurable)
- Watering halted in ESPHome when level < 5% (protects pump from dry-running)

## Consequences
- Enables: continuous level reading in HA; low-water push notification; pump dry-run protection
- Requires: one 3D-printed sensor bracket per zone (add to enclosure design brief in ADR-006)
- Requires: one-time calibration after reservoir is physically installed
- Risk: laser may behave unexpectedly on very dark or very reflective containers (clear glass and white ceramic both work well; mirror-finish metal does not)

## Kill Switch
If the VL53L0X gives unreliable readings due to reservoir material or geometry, fall back to a float switch wired to a GPIO input for a simple low-water binary alert. No firmware rewrite needed — just add a `binary_sensor` alongside the existing ToF sensor entry.
