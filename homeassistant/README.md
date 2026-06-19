# Home Assistant — Autowater

HA owns the watering *decision*; the ESP32 is a safe *executor* (ADR-008).

## Blueprint: per-plant closed-loop watering

`blueprints/automation/autowater/plant_watering.yaml`

Implements the ADR-008 stepped setpoint band: sip → settle → re-read → repeat
until `stop %`, bounded by a per-plant daily sip cap. One automation per plant.

### Install
1. Copy the blueprint into your HA config at
   `config/blueprints/automation/autowater/plant_watering.yaml`
   (or **Settings → Automations → Blueprints → Import** and point at the raw file).
2. Reload automations (or restart HA).

### Per-plant setup
For each plant:
1. **Create a Counter helper** (Settings → Devices & Services → Helpers → Counter),
   e.g. `counter.zone_a_pothos_sips`, initial value `0`. This is the daily sip
   tally; the automation resets it at local midnight.
2. **Create an automation** from the blueprint and fill in:
   - Moisture sensor — the calibrated `%` entity (e.g. `sensor.pothos_moisture`),
     **not** the raw-voltage diagnostic entity.
   - ESP32 water_plant action — `esphome.zone_a_water_plant` (Zone A) or
     `esphome.zone_b_water_plant` (Zone B). Confirm the exact name under
     **Developer Tools → Actions**.
   - Plant number — the valve index (Zone A 1–9, Zone B 1–3).
   - start % / stop % / sip seconds / settle minutes — see tuning below.
   - Daily counter — the helper from step 1; daily cap — the ceiling.
   - Flood / reservoir-low binary_sensors + watering-disable switch for this zone.

### Tuning
- **Sensitive plants** (e.g. rubber tree): small `sip seconds` + long `settle
  minutes` + low `daily cap`. The closed loop cannot overshoot because every sip
  is gated on a fresh *settled* reading.
- **Thirsty/robust plants**: bigger sips, shorter settle, higher cap.
- `sip seconds` is a **placeholder until the bench flow-rate test** (ADR-008
  assumes ~300 mL/min ⇒ 4 s ≈ 20 mL). The ESP32 hard-clamps any sip to **8 s**
  regardless, so HA can never over-deliver in a single step.

### Safety model
Interlocks are enforced **on the ESP32** — `water_plant` refuses while flood,
reservoir-low, or watering-disable is active, and flood drives all relays off
immediately. The blueprint also checks them, but only to avoid firing pointless
calls and to break the loop promptly. Losing the HA box pauses *automatic*
watering only; it can never cause a flood (that's hardware, ADR-004).
