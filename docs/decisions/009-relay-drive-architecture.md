# ADR-009: Relay Drive Architecture (Zone A GPIO expander)

## Status
Accepted

## Context
Each zone drives one relay per valve plus one for the pump (ADR-002). After assigning the other Zone A peripherals, the ESP32 runs out of safe GPIO:

- Total boot-safe, output-capable ESP32 pins (after removing the 4 input-only, 6 flash, 2 UART, and 5 strapping pins): ~15.
- Zone A already spends 8: CD74HC4067 mux (5: SIG + S0–S3), I2C for the VL53L0X (2), flood divider (1).
- That leaves **8 clean pins**, but Zone A needs **10 relays** (9 valves + pump; plant count corrected 10→9 on 2026-06-13).

So Zone A is 2 short on clean pins — and the only leftover pins are strapping/boot pins (GPIO0/2/12/15). Using them for an **active-low** relay board risks momentarily firing a valve at boot, and GPIO12 held high prevents the board booting at all. Unacceptable for a water-control system.

Zone B is unaffected: 4 relays (3 valves + pump), no mux, plenty of clean pins.

## Options Considered

| Option | Pros | Cons | Cost |
|--------|------|------|------|
| **MCP23017 I2C GPIO expander (Zone A)** | 16 outputs on the *existing* I2C bus → zero extra ESP32 pins; native ESPHome support; powers up high-Z so active-low relays stay OFF at boot | One more I2C device; relay control shares the bus with the reservoir sensor | ~$1.50 |
| Direct GPIO + strapping pins | No extra parts | Boot-glitch fires valves; GPIO12 blocks boot; fragile to flash/debug | $0 |
| 74HC595 shift register(s) | Cheap; frees pins | Still spends 3 GPIO; output-only; clunkier in ESPHome than MCP23017 | ~$1 |
| Reduce Zone A plant count to ~7 | No extra parts | Defeats the requirement (9 plants) | — |

## Decision
**Zone A: drive the 16-channel relay board through an MCP23017 I2C GPIO expander (addr 0x20) on the existing GPIO21/22 bus. Zone B: drive the 4-channel board directly from GPIO13/16/17/18.**

- No I2C address clash: VL53L0X 0x29, MCP23017 0x20.
- MCP23017 outputs power up as high-impedance inputs → the relay board's own pull-ups hold the active-low relays OFF during boot (a cleaner boot state than direct GPIO).
- All relay channels (both zones) use ESPHome `restore_mode: ALWAYS_OFF` so a reboot never re-fires a valve.
- Channel maps: see `docs/pinout.md`.

This is consistent with ADR-001's kill switch, which already names "an external I2C expander" as a contemplated path.

## Consequences
- Enables: Zone A fits 10 relays with 6 channels of headroom; ESP32 pin budget goes from "2 short" to comfortable; no strapping-pin hazards.
- Adds: one MCP23017 to the Zone A BOM (~$1.50); relay control and reservoir sensing share one I2C bus.
- Risk: if the I2C bus wedges, both reservoir sensing and relay control are affected — but ESPHome recovers the bus, and a relay-control failure de-energizes to the safe state (NC valves closed, pump off).

## Kill Switch
If the shared I2C bus proves unreliable (relay command latency or bus lockups in practice), move the MCP23017 to a second software I2C bus on spare GPIO, or fall back to a 74HC595 chain on dedicated pins. If Zone A's plant count ever drops to ≤7, direct GPIO becomes viable and the expander can be removed.
