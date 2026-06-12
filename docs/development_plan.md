# Development Plan

Last updated: 2026-06-03

Each phase proves a discrete piece of functionality with a simple bench test before the next layer is added. Phases are ordered by hardware availability — Phases 1–3 can begin immediately with parts already in hand.

---

## Parts Availability at Phase Start

| Phase | Blocking parts status |
|-------|----------------------|
| 1–3 | ✅ All parts in hand |
| 4 | ✅ All parts in hand |
| 5 | Waiting on relay boards (in transit) |
| 6 | ✅ All parts in hand |
| 7 | ✅ All parts in hand |
| 8 | Waiting on VL53L0X (in transit) |
| 9 | Post bench test (tubing) |
| 10 | Post all components received (enclosure) |

---

## Phase 1 — ESP32 + ESPHome Bring-Up

**Goal**: Flash ESPHome on the ESP32, connect to WiFi, appear in HA.

**Proves**: Board works, ESPHome toolchain operational, HA autodiscovery works, OTA updates work.

**Tests**:
- [ ] ESPHome device appears in HA with device name and version
- [ ] OTA update pushed successfully (change a name, re-flash OTA, confirm change)
- [ ] Device reconnects to HA after power cycle (no serial needed)
- [ ] Confirm zero ADC2 pins in initial pinout — ADC2 (GPIO0/2/4/12/13/14/15/25/26/27) are unusable when WiFi is active; establish this constraint in the ESPHome config from day one

**ESPHome config baseline**:
- `wifi:` + `api:` + `ota:` + `logger:`
- Set `adc_attenuation` for ADC1 pins to `11db` (0–3.3V range)
- Create `docs/pinout.md` at this stage and keep it updated as each phase assigns pins — per-project convention (see MEMORIALIZATION.md)

**Parts needed** (all in hand):
- ESP32-DEVKITC-32E ✅
- Micro-USB cable (standard)
- Computer running Home Assistant with ESPHome add-on

---

## Phase 2 — Single Soil Sensor + ADC Calibration

**Goal**: Read one capacitive moisture sensor on ADC1; calibrate it; see a stable 0–100% value in HA.

**Proves**: ADC1 reads correctly, calibration procedure works, sensor physically fits small pots.

**Tests**:
- [ ] Sensor reads a stable raw ADC value in air (~3100–3300 expected)
- [ ] Sensor reads a stable raw ADC value submerged in water (~1200–1500 expected)
- [ ] `calibrate_linear` filter maps raw → 0–100%; confirmed in HA entity
- [ ] Sensor physically fits into smallest available pot (≤8cm diameter, sensor is 16mm wide)
- [ ] Value changes meaningfully between very dry soil and freshly watered soil

**Zone B note**: Zone B (2–3 sensors) connects directly to ADC1 pins — no mux. Phase 2 covers this case. Assign Zone B sensor pins here so they're locked in before Phase 3 claims more pins. Note: Zone B sensors (3×) have not yet been ordered — order them after confirming Zone A sensor dimensions are acceptable in Phase 2.

**Parts needed** (all in hand once Dupont jumpers arrive):
- ESP32 ✅
- 1× capacitive soil sensor ✅
- Dupont jumpers ✅

> If you have any loose hookup wire on hand you can start this before the jumpers arrive.

---

## Phase 3 — Zone A: CD74HC4067 Mux + All 10 Sensors

**Goal**: Read all 10 Zone A sensors sequentially through the CD74HC4067 mux on a single ADC1 pin.

**Proves**: Mux select logic routes correctly, 3.3V VCC wiring is correct, all 10 sensors independently readable.

**Tests**:
- [ ] Mux VCC wired to ESP32 3.3V rail (NOT 5V — HC logic at 5V requires 3.5V HIGH threshold which ESP32 3.3V GPIO cannot meet)
- [ ] Each of 10 channels reads its own sensor independently (no channel bleed)
- [ ] Sensor IDs cycle correctly in ESPHome YAML (10 sensor entities in HA)
- [ ] All 10 sensors individually calibrated (air reading + water reading per sensor)
- [ ] Readings stable during active WiFi (ADC1 only — confirm no ADC2 pins used)

**ESPHome approach**: Use `custom` component or ESPHome's `cd74hc4067` support to cycle S0–S3 select pins and read `SIG` on one ADC1 pin per sensor poll.

**Parts needed** (all in hand):
- ESP32 ✅
- CD74HC4067 breakout board ✅
- 10× capacitive soil sensors ✅
- Dupont jumpers 🚚

---

## Phase 4 — Power Architecture

**Goal**: Validate the 12V → MP1584EN → 5V power chain. ESP32 and relay board powered from bench supply, not USB.

**Proves**: Buck converter stable at 5V, correct voltage delivered to ESP32 and relay logic.

**Tests**:
- [ ] MP1584EN output measured at 5.00 ± 0.1V under load (multimeter)
- [ ] 12V rail available for pump and solenoid valves
- [ ] ESP32 boots and connects to HA when powered via 5V pin (not USB)
- [ ] No instability or resets under WiFi transmission (peak ~240mA at 3.3V)
- [ ] Power flow confirmed: 12V → killswitch → pump/valve rail; 12V → buck → 5V ESP32 rail (5V rail does NOT pass through killswitch — verified in Phase 7)

**Parts needed**:
- 12V/2A wall adapter ✅
- MP1584EN buck converter ✅
- Multimeter

---

## Phase 5 — Relay Board Control

**Goal**: ESP32 GPIO switches relay channels; relay switches 12V to a solenoid valve.

**Proves**: Active-low relay logic, ESP32 3.3V GPIO sufficient to trigger both relay boards, NC solenoid behavior.

**Tests**:

**Zone A — 16-ch board (ULN2803 Darlington, active-low)**:
- [ ] Each of 11 channels (10 valves + pump) triggers on ESP32 GPIO LOW
- [ ] Relay board VCC → 5V rail; JD-VCC → 12V rail (separate power rails per board spec)
- [ ] All 11 channels audibly click and switch a test 12V LED or solenoid
- [ ] 5 spare channels verified not interfering

**Zone B — 4-ch board (optoisolated, active-low)**:
- [ ] All 4 channels trigger correctly
- [ ] JD-VCC jumper removed (separate isolated rail)
- [ ] 4 channels audibly click and switch

**Both boards**:
- [ ] NC solenoid confirmed CLOSED when relay is open (unpowered) — this is the fail-safe state
- [ ] NC solenoid opens immediately on relay close
- [ ] Pump relay channel switches 12V correctly

**Parts needed**:
- 16-ch relay board 🚚 (in transit)
- 4-ch relay board 🚚 (in transit)
- 12V power ✅ (after Phase 4)
- ESP32 ✅
- 1× solenoid valve (for test load) ✅

---

## Phase 6 — Water Delivery Bench Test *(critical milestone)*

**Goal**: Complete water flow from pump through solenoid to output. Validate valve seals, flow direction, and flow rate. This is the gate before ordering tubing and fittings.

**Proves**: Solenoid seals at low pump pressure, correct flow direction, measured flow rate, pump-before-valve sequencing.

**Tests**:
- [ ] Pump running alone (no valve open): no water exits any outlet — confirm all valves NC-sealed
- [ ] Pump ON → valve open → water exits; pump OFF → valve closed → water stops cleanly
- [ ] Valve seals under pump pressure with NO drip when closed (test for ≥30 seconds)
- [ ] Flow direction arrow on valve body identified; valve installed inlet-from-pump, outlet-to-plant
- [ ] Flow rate measured: count mL/10 seconds at nominal voltage (target: 200–400 mL/min from pump spec)
- [ ] Pump-before-valve sequencing enforced in firmware: relay sequence is `pump ON → (100ms delay) → valve OPEN → timed → valve CLOSED → (100ms delay) → pump OFF`
- [ ] Valve port OD measured physically with calipers — confirm exact OD (6.35mm = true 1/4", or metric 6mm) before ordering tubing
- [ ] Single valve open at a time confirmed (never two valves simultaneously)

**After this phase**: Order tubing, T-fittings, and bulkhead fittings based on confirmed port measurement.

**Parts needed**:
- 12V submersible pump ✅
- 1× solenoid valve ✅
- 12V power ✅ (Phase 4)
- Relay board ✅ (Phase 5)
- ESP32 ✅
- Bucket + water + short length of test tubing (any 1/4" OD PE or even a stub of the actual tubing)

> ⚠️ Pump protection: always ensure the pump is submerged in water before powering it. Running the brushless pump dry even briefly can damage the impeller. Fill the test bucket before connecting power.

---

## Phase 7 — Flood Safety System

**Goal**: XH-M131 hardware killswitch cuts 12V to pump and solenoids independently of ESP32. ESP32 also detects flood and alerts HA.

**Proves**: Hardware independence of safety path, OR logic across multiple probes, correct power rail isolation (5V ESP32 stays live).

**Tests**:

**Hardware path (independent of ESP32)**:
- [ ] Dry probes: XH-M131 relay energized, 12V flows to pump + solenoid relay board — pump runs normally
- [ ] Wet probe 1: 12V cut within milliseconds — pump stops even if ESP32 code is in an infinite loop
- [ ] Wet probe 2 (Zone A only, 3 probes): test each probe independently triggers killswitch (OR logic)
- [ ] Power failure simulation: disconnect XH-M131 power → relay de-energizes → pump cut (fail-safe)

**Power rail isolation (critical)**:
- [ ] 5V rail to ESP32 and relay logic VCC remains live after killswitch trips
- [ ] ESP32 publishes `flood_detected: true` to HA within seconds of probe contact
- [ ] HA alert fires (push notification or HA alert automation)

**Software path**:
- [ ] ESP32 GPIO interrupt detects flood and disables all relay GPIO outputs in software (belt-and-suspenders)
- [ ] After flood clears (probe dry), system requires explicit reset before watering resumes

**Parts needed**:
- XH-M131 modules ✅
- Flood sensor probes ✅
- 12V power ✅
- Pump + solenoid from Phase 6 ✅
- Relay board from Phase 5 ✅
- ESP32 ✅

---

## Phase 8 — Reservoir Level Sensing

**Goal**: VL53L0X ToF sensor reads water level as a continuous 0–100% value in HA.

**Proves**: I2C works on ESP32 (GPIO21/22), sensor reads reliably above water surface, calibration procedure.

**Tests**:
- [ ] VL53L0X appears on I2C bus scan (address 0x29)
- [ ] Sensor reads distance stably (no erratic jumps) pointed at water surface
- [ ] `calibrate_linear` filter maps sensor-to-surface distance to 0–100%
- [ ] HA entity `sensor.zone_X_reservoir_level` displays correct %
- [ ] HA automation fires when level drops below 20%
- [ ] ESPHome automation halts watering queue when level < 5% (dry-run protection)
- [ ] Verify sensor behavior over clear glass and/or the actual reservoir material — mirror-finish metal known problematic

**Parts needed**:
- VL53L0X breakout boards 🚚 (in transit)
- ESP32 ✅
- Reservoir (any bucket for testing)

---

## Phase 9 — Full Automation Logic + Offline Validation

**Goal**: End-to-end: soil moisture reading drops below threshold → watering event queued → valve sequenced → water delivered → sensor updates. Verify offline operation.

**Proves**: Watering logic correct, sequencing correct, per-plant thresholds work, system operates without internet.

**Tests**:
- [ ] Single plant: manually dry a sensor below its threshold → watering event fires (pump on, valve opens, timed pulse, valve closes, pump off)
- [ ] Two plants both dry: queue processes them sequentially, never opens two valves simultaneously
- [ ] Succulent threshold (≈20% moisture) and tropical threshold (≈50% moisture) coexist in same Zone A config without conflict
- [ ] Watering pulse delivers correct volume: measure mL output at each duration value (confirm 15–40mL range achievable)
- [ ] After watering, sensor reads higher moisture — event does not immediately re-trigger
- [ ] **Offline test**: disable WiFi router / pull Ethernet from HA server → zone continues watering autonomously based on sensor readings. HA goes unavailable but plants still get watered.
- [ ] Flood during watering cycle: flood probe triggered mid-cycle → hardware kills pump immediately, software cancels remaining queue, HA alert fires

**Parts needed**: All previous phases assembled; tubing (post Phase 6 bench test results)

---

## Phase 10 — Final Integration + Enclosure

**Goal**: Both zones complete, cable-managed, enclosed, production-ready.

**Proves**: Physical fit of all components, enclosure design correct, cable management clean.

**Tests**:
- [ ] All components fit physically inside enclosure with room for wiring
- [ ] All wire exits through cable glands (no bare holes)
- [ ] Sensor cables bundled in braided PET sleeve from enclosure exit to plant area
- [ ] Solenoid valve lines exit via 1/4" panel-mount bulkhead fittings (rear panel)
- [ ] Power cable exits via M12 cord grip
- [ ] Lid fits flush; M3 screws on bottom face only (not visible from top/front)
- [ ] Zone A and Zone B both run simultaneously for 24 hours without issues
- [ ] All HA entities reporting correctly from final installed position

**Parts needed**: Tubing + T-fittings + bulkhead fittings (after Phase 6), solid-core hookup wire, M3 screws, drip trays, 3D-printed enclosure (order after all components physically in hand per ADR-006)

---

## Functional Coverage Matrix

| Requirement / ADR constraint | Covered in phase |
|------------------------------|-----------------|
| ESP32 + ESPHome, HA autodiscovery, OTA | 1 |
| ADC2 pins avoided (WiFi conflict) | 1, 2, 3 |
| Single soil sensor + calibrate_linear | 2 |
| Sensor fits ≤8cm pots (16mm wide) | 2 |
| Zone B direct ADC (no mux) | 2 |
| CD74HC4067 VCC at 3.3V | 3 |
| Mux select routing, 10-channel poll | 3 |
| All 10 sensors individually calibrated | 3 |
| 12V → 5V buck converter stable | 4 |
| 5V rail independent from killswitch | 4, 7 |
| 16-ch relay board (Zone A), active-low, GPIO-compatible | 5 |
| 4-ch relay board (Zone B), active-low, optoisolated | 5 |
| NC solenoid safe state confirmed | 5 |
| Pump-before-valve sequencing | 6, 9 |
| Solenoid seals at low pump pressure | 6 |
| Flow direction confirmed | 6 |
| Flow rate measured (calibrate pulse duration) | 6 |
| Valve port OD measured (tubing order gate) | 6 |
| XH-M131 hardware killswitch, no ESP32 dependency | 7 |
| Flood sensor OR logic (multiple probes) | 7 |
| Flood probe fail-safe (open circuit = pump cut) | 7 |
| ESP32 flood GPIO → HA alert | 7 |
| VL53L0X I2C, distance-to-% calibration | 8 |
| HA low-level automation (20% threshold) | 8 |
| ESPHome dry-run halt (5% threshold) | 8 |
| Moisture-triggered watering (not time-based) | 9 |
| Per-plant configurable thresholds | 9 |
| Mixed succulent/tropical thresholds | 9 |
| Sequential valve delivery (one at a time) | 9 |
| Volume delivery 15–40mL achievable | 9 |
| Offline operation (watering without internet) | 9 |
| Flood during active cycle stops immediately | 9 |
| All components in enclosure, cable glands | 10 |
| Braided sleeve cable management | 10 |
| Bulkhead fittings for tubing exits | 10 |

---

## Minimum Parts to Start Phase 1

You can begin **today**.

| Part | Status |
|------|--------|
| ESP32-DEVKITC-32E | ✅ In hand |
| Micro-USB cable | Standard; almost certainly already have one |
| Computer with Home Assistant + ESPHome add-on | Assumed operational |

Phase 2 (single sensor) requires Dupont jumpers — start it as soon as they arrive, or immediately if you have any loose hookup wire on hand.
