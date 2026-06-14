# ESP32 Pin Assignments

Board: ESP32-DEVKITC-32E (38-pin, WROOM-32E module)

Last updated: 2026-06-13

## Hard Constraints

**ADC2 pins — NEVER USE when WiFi is active:**
GPIO0, GPIO2, GPIO4, GPIO12, GPIO13, GPIO14, GPIO15, GPIO25, GPIO26, GPIO27

Rationale: ADC2 is shared with the WiFi radio. Any ADC read on these pins returns garbage or errors while WiFi is connected. All analog sensors must use ADC1 (GPIO32–GPIO39).

---

## Zone A Pin Assignments

| GPIO | ADC | Function | Component | Notes |
|------|-----|----------|-----------|-------|
| GPIO32 | ADC1_CH4 | Mux SIG (analog in) | CD74HC4067 SIG pin | Single ADC input for all 10 soil sensors |
| GPIO19 | — | Mux S0 (output) | CD74HC4067 S0 | Select bit 0 (LSB) |
| GPIO18 | — | Mux S1 (output) | CD74HC4067 S1 | Select bit 1 |
| GPIO5 | — | Mux S2 (output) | CD74HC4067 S2 | Select bit 2. Strapping pin: brief glitch at boot is harmless here |
| GPIO17 | — | Mux S3 (output) | CD74HC4067 S3 | Select bit 3 (MSB) |
| GPIO33 | ADC1_CH5 | — free — | — | Freed 2026-06-12: was Pothos direct ADC (Phase 1–2); Pothos moved to mux channel C0 |
| GPIO21 | — | I2C SDA | VL53L0X | Hardware I2C bus |
| GPIO22 | — | I2C SCL | VL53L0X | Hardware I2C bus |
| GPIO23 | — | Flood sense (divider) | 10k/3.3k divider from gated 12V rail | LOW = flood (rail cut), HIGH = dry. 1k series at pin; fail-safe (broken feed reads LOW = flood). `inverted: true`. See ADR-004 |
| (I2C 0x20) | — | 16-ch relay board driver | MCP23017 expander on the GPIO21/22 bus | All Zone A relays via expander, NOT direct GPIO (insufficient clean pins after mux+I2C+flood). See ADR-009 |
| TBD | — | Status LED | Onboard or panel LED | Optional |

> **Correction 2026-06-12:** S0–S3 were previously assigned to GPIO33/34/35/36. GPIO34–36 are input-only and cannot drive the mux select lines (S0–S3 are outputs FROM the ESP32 INTO the mux — the old note had the direction backwards). Reassigned to GPIO19/18/5/17.
> GPIO34, 35, 36, 39 remain available as input-only ADC1 pins for future analog inputs. Mux EN pin ties directly to GND (active low, always enabled) — no GPIO needed.

**Zone A relay drive — MCP23017 I2C expander (ADR-009).** All relays driven via the expander (I2C addr 0x20), not direct ESP32 GPIO. 16-ch relay board is active-low; ESPHome `restore_mode: ALWAYS_OFF` on every channel so relays power up OFF. Zone A = **9 plants** (corrected from 10 on 2026-06-13): 9 valves + pump = 10 relays; 6 channels spare.

| MCP23017 ch | Relay board IN | Controls |
|-------------|----------------|----------|
| 0 | IN1 | Pump |
| 1 | IN2 | Valve 1 (Plant 1) |
| 2 | IN3 | Valve 2 (Plant 2) |
| 3 | IN4 | Valve 3 (Plant 3) |
| 4 | IN5 | Valve 4 (Plant 4) |
| 5 | IN6 | Valve 5 (Plant 5) |
| 6 | IN7 | Valve 6 (Plant 6) |
| 7 | IN8 | Valve 7 (Plant 7) |
| 8 | IN9 | Valve 8 (Plant 8) |
| 9 | IN10 | Valve 9 (Plant 9) |
| 10–15 | IN11–IN16 | Spare (headroom for more plants) |

---

## Zone B Pin Assignments

| GPIO | ADC | Function | Component | Notes |
|------|-----|----------|-----------|-------|
| GPIO32 | ADC1_CH4 | Soil sensor 1 | Capacitive sensor | Direct ADC, no mux needed |
| GPIO33 | ADC1_CH5 | Soil sensor 2 | Capacitive sensor | |
| GPIO34 | ADC1_CH6 | Soil sensor 3 | Capacitive sensor | Input-only pin |
| GPIO21 | — | I2C SDA | VL53L0X | Hardware I2C bus |
| GPIO22 | — | I2C SCL | VL53L0X | Hardware I2C bus |
| GPIO23 | — | Flood sense (divider) | 10k/3.3k divider from gated 12V rail | LOW = flood (rail cut), HIGH = dry. 1k series at pin; fail-safe (broken feed reads LOW = flood). `inverted: true`. See ADR-004 |
| GPIO13, 16, 17, 18 | — | Relay IN1–IN4 | 4-ch relay board (active-low) | Direct GPIO; `restore_mode: ALWAYS_OFF`. Valves 1–3 + pump. See ADR-009 |

**Zone B relay drive — direct GPIO (ADR-009).** Only 4 relays and plenty of clean pins (no mux in Zone B), so no expander. 4-ch board active-low; `restore_mode: ALWAYS_OFF`.

| Relay board IN | GPIO | Controls |
|----------------|------|----------|
| IN1 | GPIO13 | Valve 1 (Plant 1) |
| IN2 | GPIO16 | Valve 2 (Plant 2) |
| IN3 | GPIO17 | Valve 3 (Plant 3) |
| IN4 | GPIO18 | Pump |

---

## Reserved / Do Not Use

| GPIO | Reason |
|------|--------|
| GPIO0 | Boot mode strapping pin; ADC2 |
| GPIO2 | Boot strapping; ADC2 |
| GPIO12 | Boot strapping (MTDI); ADC2 |
| GPIO15 | Boot strapping (MTDO); ADC2 |
| GPIO6–11 | Connected to internal SPI flash — never use |
| GPIO1, GPIO3 | UART TX/RX (serial monitor / programming) |
