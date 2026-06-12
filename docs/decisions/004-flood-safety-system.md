# ADR-004: Flood Safety System

## Status
Accepted

## Context
A software bug, stuck solenoid, or runaway watering loop could overflow drip trays and damage the floor. A purely software killswitch is insufficient — if the ESP32 is the source of the bug, it cannot also be the reliable killswitch. Hardware independence is required.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **XH-M131 pre-built flood relay module** | Manufactured PCB; LED indicator; screw terminals; no perfboard assembly; fail-safe | Slightly bulkier than discrete circuit |
| Discrete NPN transistor + relay (perfboard) | Minimal components; very small | Requires perfboard soldering; harder to troubleshoot; less clean internally |
| Software-only (ESP32 GPIO interrupt) | No extra hardware | Not independent of software — fails exactly when we need it most |
| Commercial flood sensor + smart plug | Easy to source | Cloud-dependent; adds latency; adds recurring cost |

## Decision
**XH-M131 pre-built water leak detection relay module, independent of ESP32.**

The XH-M131 is a manufactured PCB that accepts water sensor probes and drives a relay output. It has an onboard LED (visible flood state at a glance), screw terminals for wiring, and requires no perfboard assembly. It goes inside the enclosure — aesthetics of the board itself don't matter, but it's cleaner than perfboard when you open the box for troubleshooting.

**Wiring:**
```
12V wall adapter ──► [XH-M131 relay NC contacts] ──► pump + solenoid relay board 12V rail
                      XH-M131 sensor inputs ◄── Flood sensors (parallel, in drip trays)
                      XH-M131 sensor inputs ──► ESP32 GPIO (shared, for HA alerting)
```

**Normal operation (dry):** Sensors dry → XH-M131 relay energized → NC contacts CLOSED → 12V flows to pump and solenoid board.

**Flood detected (wet):** Sensors detect water → XH-M131 relay de-energizes → NC contacts OPEN → 12V to pump and solenoids cut within milliseconds.

**Power failure / open-circuit sensor failure:** Relay de-energizes → pump cut. Fail-safe by default.

**Short-circuit sensor failure:** Relay always de-energized → no watering. HA alert via ESP32 GPIO allows detection.

### Flood Sensors

Resistive water detection sensors placed in drip trays. 2–3 sensors per zone, wired in **parallel** to the transistor base circuit (OR logic: any wet sensor cuts power). The same sensors also connect to ESP32 GPIO interrupt pins for HA alerting and software-side disabling of relay outputs.

### Drip Trays

Plants must sit in drip trays for flood detection to work. 2–3 grouped trays per zone recommended (not one per plant). Sensors sit flat in the tray — first water contact triggers the killswitch.

## Consequences
- Enables: hardware-independent flood protection that survives ESP32 software failures
- Enables: HA flood alerts via dual-path (hardware acts fast, software alerts user)
- Requires: drip trays for all plants (small cost, good practice regardless)
- Requires: discrete component assembly (transistor + relay + resistors); intermediate soldering skill
- Prevents: any watering while a flood condition exists (even intentional watering requires clearing the flood state)

## Kill Switch
If the transistor circuit proves unstable (false positives causing pump to cut during normal operation), replace the discrete circuit with a dedicated flood sensor relay module (e.g., XH-M131) which provides the same hardware independence with a pre-built board.
