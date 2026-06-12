# ESP32 Pin Assignments

Board: ESP32-DEVKITC-32E (38-pin, WROOM-32E module)

Last updated: 2026-06-12

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
| GPIO23 | — | Flood sensor input | XH-M131 + ESP32 GPIO | Interrupt-capable; pulled high |
| TBD | — | Relay IN1–IN11 | 16-ch relay board | 11 GPIO for valves 1–10 + pump |
| TBD | — | Status LED | Onboard or panel LED | Optional |

> **Correction 2026-06-12:** S0–S3 were previously assigned to GPIO33/34/35/36. GPIO34–36 are input-only and cannot drive the mux select lines (S0–S3 are outputs FROM the ESP32 INTO the mux — the old note had the direction backwards). Reassigned to GPIO19/18/5/17.
> GPIO34, 35, 36, 39 remain available as input-only ADC1 pins for future analog inputs. Mux EN pin ties directly to GND (active low, always enabled) — no GPIO needed.

**Relay GPIO assignments (to be filled in Phase 5):**
| Relay channel | GPIO | Controls |
|--------------|------|---------|
| CH1 | TBD | Valve 1 (Plant 1) |
| CH2 | TBD | Valve 2 (Plant 2) |
| CH3 | TBD | Valve 3 (Plant 3) |
| CH4 | TBD | Valve 4 (Plant 4) |
| CH5 | TBD | Valve 5 (Plant 5) |
| CH6 | TBD | Valve 6 (Plant 6) |
| CH7 | TBD | Valve 7 (Plant 7) |
| CH8 | TBD | Valve 8 (Plant 8) |
| CH9 | TBD | Valve 9 (Plant 9) |
| CH10 | TBD | Valve 10 (Plant 10) |
| CH11 | TBD | Pump relay |

---

## Zone B Pin Assignments

| GPIO | ADC | Function | Component | Notes |
|------|-----|----------|-----------|-------|
| GPIO32 | ADC1_CH4 | Soil sensor 1 | Capacitive sensor | Direct ADC, no mux needed |
| GPIO33 | ADC1_CH5 | Soil sensor 2 | Capacitive sensor | |
| GPIO34 | ADC1_CH6 | Soil sensor 3 | Capacitive sensor | Input-only pin |
| GPIO21 | — | I2C SDA | VL53L0X | Hardware I2C bus |
| GPIO22 | — | I2C SCL | VL53L0X | Hardware I2C bus |
| GPIO23 | — | Flood sensor input | XH-M131 + ESP32 GPIO | Interrupt-capable; pulled high |
| TBD | — | Relay IN1–IN4 | 4-ch relay board | 4 GPIO for valves 1–3 + pump |

**Relay GPIO assignments (to be filled in Phase 5):**
| Relay channel | GPIO | Controls |
|--------------|------|---------|
| CH1 | TBD | Valve 1 (Plant 1) |
| CH2 | TBD | Valve 2 (Plant 2) |
| CH3 | TBD | Valve 3 (Plant 3) |
| CH4 | TBD | Pump relay |

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
