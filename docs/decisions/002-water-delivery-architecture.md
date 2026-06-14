# ADR-002: Water Delivery Architecture

## Status
Accepted

## Context
Each zone needs to deliver small, precise volumes of water (15–40mL) to individual plants on demand. Water source is a manual-fill reservoir in the same room. Plants range from succulents (very infrequent) to tropicals (every few days). Largest pot is 10cm diameter.

Three approaches were evaluated:

## Options Considered

| Option | Pros | Cons | Cost (Zone A, 9 plants) |
|--------|------|------|------|
| **A: One peristaltic pump per plant** | Simple control (relay = pump on/off); precise volume by time; independent per plant; no manifold | 9 pumps = expensive + bulky + noisy; more failure points | ~$80–150 CAD in pumps alone |
| **B: One pump + solenoid manifold (parallel)** | Cheaper per-output than pumps; can open multiple valves simultaneously | Simultaneous watering complicates flood risk; manifold pressure balancing | ~$40–60 CAD |
| **C: One pump + solenoid manifold (sequenced, one valve at a time)** | Same low cost as B; only one plant can receive water at a time → bounded flood risk; simpler logic | Sequential only; slightly longer cycle to water all plants | ~$40–60 CAD |

## Decision
**Option C: Single submersible pump + normally-closed solenoid valves, one valve open at a time.**

For plants this small (≤10cm pots, ≤40mL per event), sequencing is not a constraint — even watering 9 plants at 30 seconds each completes in under 5 minutes. The bounded flood risk is a significant advantage: a stuck-open solenoid combined with a stuck-closed pump relay can only affect one plant's drip tray at a time before the hardware killswitch trips (ADR-004).

**Solenoid type: Normally-Closed (NC).** NC solenoids are closed when unpowered — power failure, ESP32 crash, or killswitch activation all result in all valves closing. This is the safe state.

**Pump**: Small 12V submersible pump, target flow rate 200–400 mL/min. At 300 mL/min, a 10-second pulse delivers ~50mL — appropriate for the largest tropical in the collection. Succulents get shorter pulses or less frequent watering events as configured by per-plant moisture threshold.

**Manifold**: 1/4" OD semi-rigid PE or nylon tubing (not soft silicone — solenoid valves use push-fit connectors that require semi-rigid OD). T-fittings branch from the pump outlet to each solenoid valve; short flexible drip line from valve outlet to plant.

**Pump sequencing requirement**: The selected solenoid valve (pilot-operated type, e.g., RO/appliance style) requires a minimum inlet pressure to open reliably. Pump must be energized before or simultaneously with opening a solenoid valve. ESPHome firmware must enforce this sequence: turn pump on → open valve → wait → close valve → turn pump off. Do not open valve without pump running.

## Solenoid Valve Sourcing Risk
**High risk item.** Mini 12V NC solenoid valves with barbed fittings (compatible with 4–6mm tubing) are not well-stocked at local retailers. Options:
1. **Overseas marketplace**: ~$2–4 CAD each, barbed fittings available, 3–5 week lead time
2. **NPT-threaded valves + barbed adapters**: Available at local electronics retailers, but requires sourcing adapters separately
3. **Upgrade to 1/4" NPT inline valves** (larger): easier to source, compatible with standard irrigation tubing via compression fittings

Resolution: Source solenoids from overseas during design phase (long lead time acceptable). If unavailable, fall back to NPT valves + barbed adapters from a local electronics retailer.

## Consequences
- Enables: simple relay-based control (one GPIO per solenoid); safe state on power loss; bounded per-plant flood exposure
- Prevents: simultaneous multi-plant watering (acceptable tradeoff)
- Requires: manifold plumbing assembly (intermediate skill level, soldering not required for tubing)

## Kill Switch
If solenoid valves prove unreliable (valve failure rate > 1/year), or if manifold backpressure causes uneven flow, replace solenoid manifold with individual peristaltic pumps (Option A). The ESP32 GPIO wiring is identical — only the hardware changes.
