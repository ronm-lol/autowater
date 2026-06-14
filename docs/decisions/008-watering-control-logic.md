# ADR-008: Watering Control Logic

## Status
Accepted (2026-06-14; rewritten 2026-06-13 — supersedes the earlier on-device-logic draft)

## Context
Hardware decisions (ADR-002, ADR-004) settle *what* the system can do: one pump per zone, NC solenoid valves opened one at a time with the pump running, and a hardware flood killswitch that cuts 12V independently of software. This ADR settles *how* the system decides to water, where that logic lives, and how it stays safe for moisture-sensitive plants.

Requirements:
- Soil-moisture-triggered, NOT schedule-based
- Per-plant independent sensing and output
- Internet outage must not stop watering (note: HA runs **locally** on the mini PC, so this means the LAN/HA must keep working — it does not require surviving an HA-box crash)
- One valve open at a time; pump on before any valve opens (ADR-002)
- Flood state inhibits all watering (ADR-004; hardware enforces, firmware must cooperate)
- **Moisture-sensitive plants need closed-loop start AND stop control, gentle enough never to overshoot**

**The physical constraint that shapes everything: capacitive soil sensors lag.** Water added takes minutes to percolate to sensor depth, so watering cannot be driven by a live reading — "pump until it reads 75%" keeps pumping while the number crawls up and floods the pot. Watering must be **incremental**: a small sip → wait for the reading to *settle* → re-evaluate against the settled value.

## Options Considered

| Option | Where the decision logic runs | Pros | Cons |
|--------|------|------|------|
| A: all logic in HA automations | HA | flexible, tweak in UI, HA has a real clock + database | watering stops if the HA box is down |
| B: all logic on ESP32 | ESP32 | survives HA being down | must reflash to tune; no wall clock (uptime hacks); rigid control model |
| C: ESP32 logic, HA tunes params | ESP32 | offline-safe + tunable | can't author the *logic* in HA; control model fixed in firmware |
| **D (chosen): HA decides, ESP32 safely executes** | HA automation + ESP32 primitive | author and tune all per-plant watering logic in HA; closed-loop setpoint bands; ESP32 guarantees safe execution + hardware interlocks regardless of HA | automatic watering pauses if the HA box crashes (rare; safety unaffected) |

## Decision
**Option D: Home Assistant owns the watering *decision*; the ESP32 is a safe *executor* and the authority for real-time safety.**

### Division of responsibility

**ESP32 (per zone):**
- Reads all sensors (soil via CD74HC4067 mux, reservoir via VL53L0X) and publishes them to HA
- Detects implausible sensor readings and exposes a per-plant **fault flag**
- Owns real-time safety, NOT delegated to HA:
  - **Flood:** on the flood GPIO, immediately drive ALL relay outputs LOW and refuse all watering until clear
  - **Reservoir low:** refuse watering (pump dry-run protection)
  - **Boot:** all relays `RESTORE_MODE: ALWAYS_OFF`
- Exposes exactly one watering action: **`water_plant(N, seconds)`** — performs pump-on → valve N open → run → valve N close → pump-off, `mode: single` (no overlap), **clamped to a hard maximum sip duration**, and refuses while any interlock is active

**HA (per zone):**
- Authors the per-plant watering logic and timing, fully tunable in the HA UI
- Tracks settle timing, episode state, and daily counts using HA's own clock and helper entities
- Calls the ESP32's `water_plant` action when its logic decides a sip is due
- Alerts (sensor fault, flood, reservoir low, daily-cap anomaly)

### Per-plant control model — closed-loop setpoint band, stepped

Each plant has four HA-side tunables:
- **start %** — begin a watering episode when *settled* moisture falls below this
- **stop %** — end the episode when *settled* moisture reaches this
- **sip seconds** — water delivered per step
- **settle minutes** — wait after a sip before re-reading, so decisions use settled (not lagging) moisture

Loop (per plant, in HA):
1. Settled moisture `< start%` AND no interlock AND sensor valid → start an episode
2. Call `water_plant(N, sip_seconds)`
3. Wait `settle_minutes` (no further sip to this plant)
4. Re-read: `< stop%` → another sip (step 2); `≥ stop%` → end episode

Moisture-sensitive plants use **small sips + long settles** → a gentle climb that *cannot overshoot*, because every sip is gated on a fresh settled reading. Robust thirsty plants use bigger sips + shorter settles. This is stepped rather than continuous specifically because of sensor lag (see Context) — a live "pump to target" floods the pot.

### Why HA can own the decision safely
- The flood killswitch is **hardware** (ADR-004) and the ESP32 enforces flood cutoff locally — neither depends on HA. So an HA crash can only cause a **missed watering**, never a flood.
- HA runs **locally** on the mini PC, so an **internet** outage doesn't affect it — the hard "internet outage must not stop watering" requirement still holds. Only an HA-box failure pauses *automatic* watering (rare, recoverable; manual watering via the ESP32 action still works).
- The ESP32's `water_plant` action is time-bounded (max-sip clamp), so even a buggy HA automation or a stuck command cannot run the pump indefinitely.

### Sensor fault handling — fail toward NOT watering
A disconnected sensor seen through the mux does **not** read a safe 0 V — it floats to an indeterminate voltage, possibly inside the trigger band. Therefore:
- Valid raw window: **0.5 V – 2.8 V** (V1.2 range is ~0.95 V wet to ~2.2 V dry; window gives headroom without admitting rail-ish float values)
- A sensor is promoted to FAULTED after **3 consecutive** out-of-window readings (rejects mux switching transients); it auto-clears after **3 consecutive** in-window readings
- The ESP32 exposes a per-plant fault `binary_sensor`; HA waters only on valid readings
- **Commissioning:** measure each sensor's dry and wet voltage as it's installed — confirms the unit works, sets its per-plant calibration, and verifies the 0.5–2.8 V fault window comfortably brackets its real range (widen the window if any sensor falls outside)

### Interlocks (ESP32-enforced, exposed to HA)

| Interlock | Source | Behavior |
|-----------|--------|----------|
| Flood detected | GPIO23 LOW (divider on gated rail, ADR-004) | ESP32 **immediately drives all relay outputs LOW** and refuses watering; exposed to HA for alerting. The relay board's 5 V logic isn't cut by the killswitch, so firmware must clear its own outputs — otherwise the pump re-energizes the instant the hardware relay restores 12 V (restart-after-flood runaway) |
| Reservoir low | VL53L0X distance > threshold | ESP32 refuses sips (pump dry-run protection); HA alerts. Threshold set after reservoir purchase |
| Global disable | HA switch entity (ESP32 also honors) | user-level master off |

### Safety caps
- **Max sip duration** — a hard ESP32 clamp on every `water_plant` call (e.g. 8 s ≈ 40 mL at the ADR-002 nominal 300 mL/min). No single sip can over-deliver regardless of what HA requests.
- **Per-plant daily sip cap** — HA-side, using HA's clock, **configurable per plant**. Once a plant reaches its daily sip limit, no further automatic watering for that plant until the daily reset. This hard-bounds a non-converging plant (stuck sensor, clogged emitter, hydrophobic soil, popped tube): instead of sipping endlessly it is capped at (daily cap × max sip) per day. Sensitive plants get a low cap (e.g. 3); robust thirsty plants get more. No alert — a silent hard ceiling (accepted; plants are monitored by eye).
- Measure actual pump flow rate at the bench test and update the sip↔mL mapping and the max-sip clamp.

### Known undetected failure mode (accepted for v1)
A mechanically stuck-open valve or welded relay contact cannot be detected — there is no flow or valve-position feedback. With the pump off, a stuck-open NC valve passes no water (the pump is the only pressure source), so the realistic exposure is over-delivery during a sip. Backstops: the max-sip clamp, the per-plant daily sip cap, and ultimately drip tray → hardware killswitch (ADR-004). A flow sensor is the v2 path.

### Explicitly deferred (not in v1)
- **Quiet hours** — easy for HA to add now that it owns timing, but defer until the system is lived-with.
- **Flow sensing / volume verification** — see Kill Switch.
- **Weather / seasonal modulation** — a natural future benefit of HA owning the decision; not v1.

## Consequences
- Enables: author and tweak every per-plant watering rule in the HA UI; closed-loop start/stop setpoint bands; gentle stepped watering that's safe for moisture-sensitive plants; HA's clock and database remove the on-device uptime/persistence hacks the earlier draft needed
- Requires: per-plant HA automations (or one reusable blueprint); ESP32 firmware for the safe `water_plant` action + interlocks + sensor/fault exposure; bench flow-rate measurement; the flood divider + flyback diodes (ADR-004)
- Prevents: overshoot on sensitive plants (stepped sips + settle); watering during flood, low reservoir, or on a faulted sensor (all ESP32-enforced); pump runaway (max-sip clamp)
- Tradeoff accepted: automatic watering pauses if the **HA box** is down (not the internet — HA is local). Flood safety is hardware and unaffected; manual watering remains available.

## Kill Switch
- If HA-box reliability proves inadequate (frequent crashes leaving plants unwatered), move the decision loop onto the ESP32 (toward Option C). The ESP32's safe `water_plant` primitive and interlocks stay put — only the decision relocates.
- If the stepped closed-loop is too slow in practice for some plants, raise their sip size / shorten their settle.
- If time-based sips drift with pump age or clogging, add an inline flow sensor and switch to volume-based sips.
