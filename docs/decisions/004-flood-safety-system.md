# ADR-004: Flood Safety System

## Status
Accepted

## Context
A software bug, stuck solenoid, or runaway watering loop could overflow drip trays and damage the floor. A purely software killswitch is insufficient — if the ESP32 is the source of the bug, it cannot also be the reliable killswitch. Hardware independence is required.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **XH-M131 pre-built flood relay module** | Manufactured PCB; LED indicator; screw terminals; no perfboard assembly | Energize-on-water (not fail-safe against its own failure — see Failure Modes); slightly bulkier than discrete circuit |
| Discrete NPN transistor + relay (perfboard) | Minimal components; very small | Requires perfboard soldering; harder to troubleshoot; less clean internally |
| Software-only (ESP32 GPIO interrupt) | No extra hardware | Not independent of software — fails exactly when we need it most |
| Commercial flood sensor + smart plug | Easy to source | Cloud-dependent; adds latency; adds recurring cost |

## Decision
**XH-M131 pre-built water leak detection relay module, independent of ESP32.**

The XH-M131 is a manufactured PCB that accepts water sensor probes and drives a relay output. It has an onboard LED (visible flood state at a glance), screw terminals for wiring, and requires no perfboard assembly. It goes inside the enclosure — aesthetics of the board itself don't matter, but it's cleaner than perfboard when you open the box for troubleshooting.

**Wiring:**
```
12V adapter ──┬─► [XH-M131 NC contacts] ──► GATED 12V rail ──► pump + relay board JD-VCC (coil supply)
              │                                  │
              │                                  └─► [10k] ──┬── [1k] ──► ESP32 GPIO23   (~3V when rail present)
              │                                            [3.3k]
              │                                              │
              │                                             GND
              └─► buck converter ──► 5V (ALWAYS ON) ──► ESP32 + relay board logic VCC

XH-M131 sensor (CDS) input ◄── flood probes (parallel, in drip trays; any wet → relay energizes → NC opens → rail cut)
```

**Board behavior (confirmed by bench test 2026-06-12/13):**
- LEDs: RED = power present; BLUE = relay energized
- Screw terminals (unlabeled, numbered 1/2/3 left→right): **2 = COM, 1 = NC** (closed when dry/de-energized), **3 = NO** (closed when wet/energized)
- **The module is ENERGIZE-ON-WATER**: the relay is *de-energized when dry* and *energizes when it detects water*. Pump 12V is therefore routed through the **NC pair (terminals 1–2)**, which is closed when dry.

**Normal operation (dry):** Relay de-energized → NC (1–2) CLOSED → 12V flows to pump and solenoid board.

**Flood detected (wet):** Relay energizes → NC (1–2) OPENS → 12V to pump and solenoids cut within milliseconds.

> **Correction 2026-06-13:** Earlier text claimed the relay was *energized when dry* and that power loss / broken probe would de-energize and cut the pump ("fail-safe by default"). Bench testing proved the opposite energize direction. The fail-safe claims were wrong; the true behavior and its limits are documented below. The NC wiring choice itself was correct.

### Failure Modes (honest analysis)

This module **catches a real flood** — water present, everything in the chain working — which is the realistic scenario (stuck valve / runaway pump → tray fills → probe trips → pump cut). It is **not fail-safe against failures of the safety system itself**, because cutting the pump requires the board to be alive, powered, and actively detecting:

| Event | Result | Safe? |
|-------|--------|-------|
| Actual flood, system healthy | Relay energizes, pump cut | ✓ |
| Total 12V power loss | Pump dead anyway (shared supply) | ✓ (by shared supply, not relay logic) |
| Broken/loose probe wire | Reads dry → relay stays de-energized → pump runs → later floods uncaught | ✗ silent loss of protection |
| XH-M131 internal fault (dead comparator/relay) | Likely fails de-energized → pump runs → floods uncaught | ✗ (backstopped by ESP32 software path) |
| Probe short / always-wet | Relay energizes → pump cut permanently | Fails toward no-watering (annoying, not dangerous) |

**Backstops:**
- The independent **ESP32 software path** reads the same probes and can cut the pump via its own relay board — covering the dead-XH-M131 case (so long as the ESP32 and probes are healthy).
- The broken-probe case is the genuine hole: a resistive water probe **cannot distinguish "dry" from "disconnected"** (both are open circuit), so a loose wire silently disables flood protection on *both* the hardware and software paths. There is no electrical detection for this — it is covered by the mandatory periodic test below.

### Mandatory Periodic Fail-Safe Test (annual, per zone)

Because silent failures of the safety chain are undetectable electrically, both zones' flood systems **must be physically tested at least once per year** (calendar reminder; pick a fixed date e.g. start of each year):

1. With the system powered and idle, pour water into each drip tray until a probe is wetted.
2. Confirm the pump's 12V is cut (pump cannot run; BLUE LED on; NC contacts open) **and** the ESP32 reports the flood in HA.
3. Repeat for every probe individually (verifies each parallel probe and its wiring — this is what catches a loose probe wire).
4. Dry everything; confirm the system returns to normal (relay releases, watering re-enabled).
5. Log the test date. A probe that fails to trip indicates a broken wire or a failed board → fix before relying on the system again.

This test is the only thing that catches the broken-probe and dead-board failure modes. It is not optional.

### Flood Sensors

Resistive bare-metal-contact water probes placed in drip trays. Zone A: 3 probes (one per shelf tray); Zone B: 1 probe. Per zone, probes wire in **parallel** to the XH-M131 sensor (CDS) input (OR logic: any wet probe trips the relay). Probe must be bare-metal-contact (a teardrop leak probe) — NOT a photoresistor or resin-coated comb (see hardware notes; a coated probe can't detect standing water).

**ESP32 flood sensing — voltage divider on the gated rail (decided 2026-06-13; revised from an optocoupler after a complexity review):** The ESP32 does NOT tap the probe line (12V-referenced — unsafe for a 3.3V GPIO — and the relay is a single SPDT already used for the pump). Instead a **resistor divider** on the **gated 12V rail** (10k from rail to node, 3.3k node to GND → ~3.0V; 1k series from node to GPIO23 for clamp protection) shifts rail-presence onto the GPIO. Rail present (dry) → GPIO23 ~3V = HIGH; flood cuts the rail → GPIO23 pulled to 0V through the lower resistor = LOW = flood. **Fail-safe:** a broken 12V feed also pulls GPIO23 LOW = read as flood = ESP32 stops watering.

*Why not an optocoupler:* galvanic isolation is unnecessary here — the 12V rail and the ESP32 share one supply and one ground (the buck converter hangs off the same 12V adapter). There is no separate ground domain or mains to isolate from, so an opto would add an IC and an isolation concept for no real benefit. A divider is the proportionate level-shift.

This **mirrors** the hardware verdict (not an independent second detector — accepted; the XH-M131 is the authoritative killswitch and the ESP32's role here is HA alerting + watering interlock per ADR-008). It also lets the ESP32 detect that the killswitch cut the pump, so it won't keep commanding a dead pump.

**Inductive loads:** the gated rail switches solenoid coils and the pump motor. Fit a **flyback diode (e.g. 1N4007) across each solenoid and the pump** — this protects the relay contacts from back-EMF *and* keeps the rail clean enough for the divider. See BOM item 6c.

### Drip Trays

Plants must sit in drip trays for flood detection to work. 2–3 grouped trays per zone recommended (not one per plant). Sensors sit flat in the tray — first water contact triggers the killswitch.

## Consequences
- Enables: hardware-independent flood protection that survives ESP32 software failures, for the realistic flood case (healthy system, water present)
- Enables: HA flood alerts via dual-path (hardware acts fast, software alerts user, and software backstops a dead XH-M131)
- Requires: drip trays for all plants (small cost, good practice regardless)
- Requires: only screw-terminal wiring — the XH-M131 is a pre-built module, no perfboard assembly or soldering
- Requires: **annual physical fail-safe test of every probe in both zones** — the only detection for broken-probe / dead-board silent failures
- Prevents: any watering while a flood condition exists (even intentional watering requires clearing the flood state)
- Does NOT provide: true fail-safe behavior against failures of the safety hardware itself (energize-on-water module). A broken probe wire silently disables protection on both hardware and software paths. Accepted for this build given the dual-path design, shared-supply power-loss coverage, and the annual test; revisit if true fail-safe is wanted (see Kill Switch)

## Kill Switch
Two distinct triggers to revisit this decision:

1. **Instability:** if the XH-M131 gives false positives (cutting the pump during normal operation) or fails to trigger on wet probes after exhausting sensitivity-pot adjustment, fall back to a discrete NPN transistor + relay circuit on perfboard, fully under our control.
2. **True fail-safe wanted:** if the energize-on-water limitation proves unacceptable (e.g. an uncaught flood causes real damage, or the annual-test discipline isn't sustainable), switch to an **energize-on-dry topology**: a relay held energized during healthy dry operation and wired through its NO contact, so that water, power loss, broken wire, OR a dead board all de-energize → cut. This is the textbook fail-safe killswitch and inverts the current module's behavior; it requires either a different module or a small relay-driver circuit, and is the correct path if flood consequences are judged severe.

> **Edit 2026-06-12:** Consequences and Kill Switch sections originally described the discrete transistor circuit from a draft prior to the XH-M131 decision; updated to match the accepted decision.
> **Edit 2026-06-13:** Bench testing revealed the XH-M131 is energize-on-water, contradicting the original fail-safe claims. Wiring/behavior corrected, true failure modes documented, annual fail-safe test mandated, and an energize-on-dry escape hatch added to Kill Switch. The decision to use the XH-M131 stands, but its safety properties are now documented accurately rather than aspirationally.
