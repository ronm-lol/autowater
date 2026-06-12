# ADR-008: Watering Control Logic

## Status
Proposed

## Context
Hardware decisions (ADR-002, ADR-004) settle *what* the system can do: one pump per zone, NC solenoid valves opened one at a time with the pump running, and a hardware flood killswitch that cuts 12V independently of software. This ADR settles *how* the system decides to water: where the control logic lives, what triggers a watering event, how much water is delivered, and which conditions inhibit watering.

Hard requirements inherited from project constraints:
- Soil-moisture-triggered, NOT schedule-based
- Per-plant independent sensing and output
- Must operate fully offline — an internet or HA outage must not stop watering
- One valve open at a time; pump on before any valve opens (ADR-002)
- Flood state inhibits all watering (ADR-004; hardware enforces, firmware must also respect)

## Options Considered

| Option | Pros | Cons | Cost |
|--------|------|------|------|
| **A: All logic in Home Assistant automations** | Rich UI; easy iteration; no firmware changes to tune | Violates offline requirement — HA VM down = plants die quietly; latency; two sources of truth | $0 |
| **B: All logic on-device (ESPHome), HA monitor-only** | Survives any HA/network outage; single source of truth; deterministic | Tuning requires OTA flash for every threshold change | $0 |
| **C: Logic on-device, parameters adjustable from HA** | Offline-safe like B; thresholds/doses tunable live via HA number/switch entities, persisted to flash on the device | Slightly more firmware complexity (globals + restore_value entities) | $0 |

## Decision
**Option C: all control logic runs in ESPHome firmware on each zone's ESP32. HA is a monitoring and tuning surface, never a dependency.**

### Time base — no wall clock dependency

All durations (lockouts, caps, timestamps) are measured in **seconds of device uptime** (ESPHome uptime sensor; 32-bit seconds, no practical rollover). The system never depends on SNTP or HA-provided time, which would break offline. Consequence: runtime state (lockout timers, daily counters) **resets on reboot** — see Boot behavior for why this is safe.

### Parameter persistence

Only user-set parameters persist to flash (NVS): per-plant threshold and dose `number` entities and the global enable `switch`, all with `restore_value: true` and explicit `initial_value` in YAML so a blank-flash device is safe out of the box. These are written only on user change — no flash-wear concern. All runtime state (timers, counters, queue) is ephemeral by design.

### Boot behavior

- All relay GPIOs (pump + valves): `RESTORE_MODE: ALWAYS_OFF` — every boot starts with pump off, valves closed, regardless of prior state.
- A **boot lockout** (default = soak lockout duration, 60 min) applies to all plants on every boot. This makes reboots conservative: a brownout loop or repeated OTA cannot deliver repeated doses, even though runtime counters reset.

### Control loop (per zone)

Every **15 minutes**, evaluate all plants and take a **snapshot** of eligibility (the queue is not re-evaluated mid-cycle):

1. A plant is **eligible** when ALL of:
   - Calibrated moisture below its per-plant threshold (default 30%)
   - Not in soak lockout (default 60 min of uptime since that plant's last watering — water needs time to percolate to sensor depth)
   - Rolling cap not reached (default 3 events per plant per 24 h of uptime; counter resets on reboot, which the boot lockout makes safe)
   - Sensor not FAULTED (see Sensor fault handling)
   - No zone interlock active (see Interlocks)
2. Eligible plants are serviced sequentially:
   - Pump ON → wait 1 s (pressure) → valve N OPEN → run plant N's dose seconds → valve N CLOSE → wait 2 s (manifold pressure settle) → next valve, pump stays on
   - After the last plant's valve closes: wait 1 s → pump OFF
   - Brief deadheading between valve switches is harmless for a small submersible centrifugal pump at <5 PSI
3. A **hard per-cycle runtime cap** aborts the queue (valves closed, then pump off) if total pump-on time exceeds it. Derivation, Zone A: 10 plants × 8 s max dose + 9 × 2 s gaps + 1 s pre-run = 99 s; ×1.2 margin ≈ **120 s**. Zone B equivalent: 3 × 8 + 2 × 2 + 1 = 29 s; cap **40 s**. This is a logic-bug backstop independent of, and beneath, the hardware flood killswitch.

### Dosing

Time-based: per-plant dose in **seconds** via a `number` entity bounded **1–8 s, step 1** (at the ADR-002 nominal 300 mL/min, 8 s = 40 mL = the ADR-002 maximum dose; the bound enforces the spec). `initial_value`: 6 s (~30 mL) for tropicals, 3 s (~15 mL) for succulents. **Measure the actual flow rate at the bench test and update both this paragraph and the bound if it differs materially from 300 mL/min.** No flow sensor in v1 — dosing self-corrects via the moisture trigger, and the cap alert flags chronic over-delivery.

### Sensor fault handling — fail toward NOT watering

A disconnected sensor seen through the CD74HC4067 does **not** read a safe 0 V — a missing sensor leaves the selected channel high-Z and GPIO32 floats to an indeterminate voltage, potentially inside the trigger band. Therefore:

- Valid raw window: **0.5 V – 2.8 V** (V1.2 range is ~0.95 V wet to ~2.2 V dry; the window gives headroom without admitting rail-ish float values)
- A sensor is promoted to FAULTED only after **3 consecutive** out-of-window readings (rejects mux switching transients); it auto-clears after **3 consecutive** in-window readings but the plant stays ineligible until the next 15-min evaluation
- FAULTED plants are excluded from automatic watering and exposed as a per-plant fault `binary_sensor` for HA alerting
- Manual "water now" for a FAULTED plant works — the fault override bypasses **only the sensor eligibility check**, never interlocks, lockouts, or caps

### Interlocks — any one inhibits ALL watering in the zone, including "water now"

| Interlock | Source | Behavior |
|-----------|--------|----------|
| Flood detected | Flood GPIO (GPIO23) | **Immediately drive ALL relay outputs LOW (valves then pump), abort queue, inhibit, alert.** Firmware must not rely on the hardware 12V cut: the relay board's 5V logic rail is not cut by the killswitch, so a GPIO left HIGH would re-energize the pump the instant the hardware relay restores 12V — a restart-after-flood runaway. Firmware clears its outputs; watering resumes only via a fresh evaluation cycle after the flood clears |
| Reservoir low | VL53L0X distance > threshold | Inhibit new cycles (pump dry-run protection); alert. Threshold set after reservoir purchase |
| Global disable | HA switch entity (persisted) | User-level master off |

**Flood GPIO active level: TBD at bench.** ADR-004 wires the probe line to both the XH-M131 input and GPIO23; the resting level depends on the module's probe bias. Verify with a multimeter during the flood bench test and record here (assumption to verify: dry = HIGH with pull-up, wet = LOW).

### Known undetected failure mode (accepted for v1)

A mechanically stuck-open valve or welded relay contact cannot be detected — there is no flow or valve-position feedback. With the pump off, a stuck-open NC valve passes no water (pump is the only pressure source), so the realistic exposure is over-delivery during a cycle. Backstops: per-cycle cap, rolling cap alert, and ultimately drip tray → hardware killswitch (ADR-004). A flow sensor is the v2 path (see Kill Switch).

### HA surface (per zone)

- Per plant: moisture %, raw voltage (diagnostic), threshold (number), dose seconds (number, 1–8 s), sensor-fault flag, last-watered (uptime-relative), "water now" button (respects interlocks, lockouts, caps)
- Zone: global enable switch, reservoir level, flood alert, watering-in-progress indicator, cap-exceeded alert (a plant repeatedly hitting its cap signals a leak, dead valve, or sensor drift)

### Implementation notes (ESPHome reality)

ESPHome YAML cannot build a runtime list and iterate it: the "queue" is implemented as a **static unrolled sequence** — one guarded block per plant inside a single `script` with `mode: single` (no re-entrancy), each block checking its plant's eligibility snapshot before acting. Uptime-seconds comparisons need small lambdas; this is the accepted scope of C++ in this project (single-expression conditions, no custom components).

### Explicitly deferred (not in v1)

- **Quiet hours** — needs wall-clock time; offline-fragile. Revisit after living with the system.
- **Flow sensing / volume verification** — see Kill Switch.
- **Multi-dose soak cycles** — the soak lockout approximates this at lower complexity.

## Consequences
- Enables: plants survive indefinite HA/internet outages; live tuning from HA without reflashing; bounded worst-case water release per cycle (cap × flow ≈ 600 mL Zone A) beneath the hardware killswitch
- Requires: per-plant commissioning pass (threshold + dose); bench measurement of pump flow rate; bench verification of flood GPIO polarity
- Prevents: automatic watering on faulted sensors, during floods, with a low reservoir, or in rapid succession after reboots — the system fails dry, never wet
- Risk remaining: time-based dosing drifts with pump age/tubing clogging (mitigated by moisture-trigger self-correction + cap alert); stuck-open valve undetectable in v1 (accepted above)

## Kill Switch
- If time-based dosing proves inaccurate in practice (consistent over/under-watering at stable dose settings), add an inline flow sensor and switch to volume-based dosing.
- If sensor faults flap (repeated promote/clear cycles in HA logs), widen the debounce count or revisit the valid window — persistent flapping means the 5 ms mux settling delay needs raising.
- If parameter tuning via HA number entities proves too fiddly, fall back to Option B with a YAML substitutions block per plant (tuning via OTA flash).
- If uptime-based caps prove confusing in practice (counters resetting on reboot), add HA-provided time with explicit offline fallback — but only with evidence the simple model actually misbehaves.
