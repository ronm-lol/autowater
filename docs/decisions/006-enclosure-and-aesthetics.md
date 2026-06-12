# ADR-006: Enclosure Design and Aesthetics

## Status
Accepted

## Context
The system is semi-visible in a living space. The user's aesthetic direction is minimal/modern: clean lines, no exposed wires, matte finish. Generic off-the-shelf ABS project boxes are not acceptable. Reservoirs are separate attractive containers chosen by the user — the electronics enclosure is the design challenge.

The soil sensor cables (up to 10 in Zone A) are the hardest aesthetic problem: 10 thin cables running from the enclosure to individual pots will look messy without deliberate cable management.

## Options Considered

| Option | Pros | Cons | Cost |
|--------|------|------|------|
| **Custom 3D-printed enclosure (ordered print)** | Can look like commercial product; tailored to component layout; matte finish achievable | Requires CAD design; lead time 1–2 weeks; one-shot (hard to iterate) | ~$20–40 CAD printed |
| Generic ABS project box | Cheap; available immediately | Looks hobbyist; visible seams; usually grey or beige | ~$8–12 CAD |
| Aluminum enclosure (Hammond) | Very premium look; durable | Expensive; harder to drill/modify; more complex to work with | ~$30–60 CAD |
| Wooden enclosure (laser-cut or handmade) | Warm, natural; pairs well with plant aesthetic | Doesn't match minimal/modern direction; harder to seal against humidity | ~$20–40 CAD |

## Decision
**Custom 3D-printed enclosure, ordered from an online 3D print service or local makerspace.**

Design in white or light grey PETG. Professional print services achieve significantly better surface finish than home printers. PETG preferred over PLA for humidity resistance.

### Enclosure Design Spec

**Zone A enclosure** (larger — must fit 16-ch relay board):
- Target external dimensions: ~200 × 120 × 60mm
- Internal layout: relay board on base; ESP32, LM2596, XH-M131 on one side
- Lid: flush-fit, countersunk M3 screws on bottom face (not visible from top/front)
- Front face: one small rectangular LED window (3mm) for status indicator; optionally a small OLED cutout
- Rear face: all connections — power barrel jack, cable glands for sensor bundles, cable glands for solenoid valve wires
- No exposed wires on front or top

**Zone B enclosure** (smaller — 4-channel relay board):
- Target external dimensions: ~120 × 80 × 50mm
- Same design language as Zone A

**Cable glands:** M12 nylon cable glands (~$0.50 each) for all wire exits. These provide a clean, professional finish where cables exit the enclosure.

### Soil Sensor Cable Management

This is the primary aesthetic challenge. 10 sensor cables from the enclosure to 10 pots will look messy without management.

**Mitigation strategy:**
1. **Bundle from enclosure to plant area**: Route all sensor cables together in a braided cable sleeve (PET expandable sleeving, black or white) until they reach the plant zone. The sleeve exits through one large cable gland on the enclosure rear.
2. **Fan out near plants**: Sleeve opens up near the plants; individual cables route to pots with adhesive cable clips along shelf edges.
3. **Thin cables**: Specify sensors with thin cables (< 2mm diameter). Standard V1.2 sensors have ~2mm cables — acceptable.
4. **Colour**: White or black cables both work for minimal/modern; match to shelf colour.

Solenoid valve cables (shorter, run to drip manifold) bundle similarly.

### Reservoir
User selects the container. Any food-safe vessel works: glass terrarium, ceramic planter, clear acrylic box. The pump is fully submerged. Tubing exits from the top. The electronics enclosure sits adjacent.

For a clean look: the reservoir lid (if any) should have a single neat hole for the pump power cable and outlet tubing — or the tubing/cable simply exits over the rim with a cable clip holding it flat.

## Consequences
- Enables: professional appearance; enclosure tailored to component layout; no exposed wires
- Requires: CAD design of enclosure (Fusion 360 or FreeCAD); 1–2 week print lead time; cable gland sourcing
- Delays: enclosure design happens after component selection is finalised (can't design the box until you know what goes in it)
- One risk: if component layout changes after the enclosure is designed, the print must be re-ordered. Design the enclosure last, after all components are physically in hand.

## Kill Switch
If 3D printing lead time becomes a problem or the design isn't working, fall back to a standard aluminum project enclosure (~$25–40 CAD). Aluminum can be powder-coated matte white or black for a premium look, but requires drilling for ports.
