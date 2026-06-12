# ADR-005: Power Architecture

## Status
Accepted

## Context
Each zone unit requires:
- 12V DC for submersible pump (~2–3W) and solenoid valves (~1W each, one at a time)
- 5V DC for ESP32 and relay board logic (~0.5–1W)
- Mains power is available at both zone locations
- No battery backup requirement

## Options Considered

| Option | Pros | Cons | Cost |
|--------|------|------|------|
| **Single 12V adapter + LM2596 buck converter** | One wall adapter per zone; clean; efficient step-down; standard approach | Buck converter requires assembly (or pre-built module) | ~$15–20 CAD total |
| Separate 12V adapter (pump/valves) + 5V USB adapter (ESP32) | No buck converter needed; simple | Two wall plugs per zone = 4 plugs total for two zones; messy | ~$20 CAD + outlet real estate |
| USB-C PD trigger + buck converter | Modern; single cable | More complex sourcing; PD negotiation adds a failure point | ~$20–25 CAD |

## Decision
**Single 12V/2A wall adapter per zone + LM2596-based buck converter module (12V → 5V).**

Power flow:
```
Wall outlet
  └─► 12V/2A adapter
        ├─► Hardware killswitch relay (ADR-004)
        │     └─► 12V to pump + solenoid relay board
        └─► LM2596 buck converter
              └─► 5V to ESP32 + relay board logic VCC
```

The relay board 12V rail passes through the killswitch; the 5V logic rail does not. This ensures the ESP32 stays powered after a flood event (so it can publish the HA alert) while the pump and solenoids are cut.

**LM2596 module**: Pre-built adjustable buck converter (~$3–8 CAD). Set output to 5.0V using onboard trimpot before first use. Efficiency ~85% at this load. Alternatively, a fixed 5V LM2596S-5.0 module eliminates the trimpot step.

## Power Budget

| Component | Voltage | Peak current | Peak power |
|-----------|---------|-------------|------------|
| ESP32 (WiFi peak) | 3.3V | 240mA | 0.8W |
| Relay board logic | 5V | 50mA | 0.25W |
| 1× solenoid valve (open) | 12V | ~83mA | ~1.0W |
| Submersible pump | 12V | ~200mA | ~2.4W |
| LM2596 buck (5V @ 0.3A, 85% eff.) | 12V | ~35mA | ~0.42W |
| **Total peak (active watering)** | | | **~4.9W** |
| **Total idle (no watering)** | | | **~1.0W** |

12V/2A = 24W → 4.9× headroom over peak. A 1A adapter would also work (4.9W < 12W) with 2.4× headroom, but 2A recommended for margin.

## Consequences
- Enables: single wall plug per zone; clean power distribution; ESP32 survives killswitch event for alerting
- Prevents: powered operation without mains (no battery/solar path — acceptable per requirements)
- Risks: LM2596 trimpot adjustment requires multimeter for initial setup; skip this by ordering fixed-5V variant

## Kill Switch
If the LM2596 module runs hot or proves unreliable, replace with a dedicated 12V→5V DC-DC converter with fixed output (e.g., Murata OKI-78SR series) — these are more efficient and require no adjustment, but cost ~$5–10 more.
