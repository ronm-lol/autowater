# Requirements

## Changelog
- **2026-05-21** — Initial requirements gathered from user
- **2026-05-21** — Aesthetics requirements added; enclosure approach revised

---

## Environment
- **Location**: Indoor only
- **Zones**: 2 decentralized zones (one unit per zone, each with own reservoir)
  - Zone A: ~9 plants (mixed succulents/cacti + tropicals/houseplants)
  - Zone B: 2–3 plants (includes rubber tree; consolidated from original zones 2 and 3)
- **Pot sizes**: Smallest pots are 8–12cm (3–5") diameter

## Water Source
- **Type**: Reservoir/tank (manual-fill; no plumbing connection)
- **Location**: Same room as plants; water source is >2m from some plants within each zone
- **Architecture**: Each zone has its own dedicated reservoir

## Power
- **Availability**: Mains power (wall outlet) available at both zone locations
- **Cable routing**: No stated constraints

## Control & Connectivity
- **Remote monitoring**: Not required
- **Home automation**: Home Assistant integration desired
- **Offline requirement**: System must continue watering autonomously during internet outages — all watering decisions made locally on-device
- **UI**: HA integration for visibility and alerts is sufficient; no custom web dashboard needed

## Watering Logic
- **Trigger**: Soil-moisture-based (hygrometer readings), not time-schedule-based
- **Per-plant control**: Each plant needs an independently controllable water output
- **Per-plant sensing**: Each plant needs its own soil moisture sensor
- **Implication**: Zone A requires ~9 sensors + ~9 independent water outputs

## Hardware & Skills
- **Existing hardware**: None — starting from scratch
- **Electronics skill**: Intermediate (comfortable with soldering)
- **3D printing**: No printer; can order prints from online services

## Budget & Scope
- **Budget ceiling**: $250–$500 CAD total (includes shipping to Canada)
- **MVP**: Both zones working simultaneously — not phased
- **Currency note**: All costs must include shipping to Canada; note USD→CAD conversion where applicable

---

## Aesthetics
- **Visibility**: Semi-visible — in living space, seen daily
- **Direction**: Minimal / modern — clean lines, matte finish, no exposed wires
- **Enclosure**: Custom 3D-printed (ordered from print service); PETG, matte white or grey
- **Reservoir**: Separate attractive container chosen by user (glass, ceramic, acrylic); electronics enclosure sits adjacent
- **Cable management**: Sensor cables bundled in braided PET sleeve from enclosure to plant area; individual cables fanned out with adhesive cable clips along shelf edges
- **Connections**: All ports on rear face of enclosure; cable glands at all wire exits; no visible screws on top or front

## Water Delivery Architecture
- **Approach**: Single pump per zone + normally-closed solenoid valves, one valve open at a time (sequenced/Option C)
- **Volume per event**: Small — 15–40ml per plant (succulents less; tropicals more)
- **Solenoid type**: Normally-closed (safe state = closed; power failure stops all flow)

## Flood Safety System
- **Flood sensors**: Resistive water detection sensors placed in drip trays under plants
- **Tray strategy**: 2–3 grouped drip trays per zone (not one per plant) to keep sensor count manageable
- **Hardware killswitch**: Flood sensors wire to a hardware relay in series with pump 12V power supply
  - Relay coil continuously energized during normal operation (contacts closed = pump can run)
  - Water detected → coil de-energizes → contacts open → pump power cut
  - Relay coil power failure → pump power also cut (fail-safe by default)
  - **Independent of ESP32 software** — a software bug cannot prevent this from triggering
- **Software layer**: Same flood sensors also wired to ESP32 GPIO for HA alerts ("FLOOD DETECTED")
- **OR logic**: Any wet tray triggers killswitch — sensors wired in parallel to relay control circuit

## Open Questions / Risks
1. **Sensor size vs. small pots**: Smallest pots are 8cm. Standard capacitive sensors (~3.5cm wide) should fit, but need to verify specific sensor dimensions before purchasing.
2. **Reservoir level sensing**: Assumed wanted (low-water HA alert) — confirm before finalizing design.
3. **Mixed thresholds within Zone A**: Succulents and tropicals have very different moisture thresholds. System must support per-plant configurable thresholds, not one global value per zone.
4. **Tray procurement**: User has no drip trays currently. Will need to source appropriate grouped trays.
