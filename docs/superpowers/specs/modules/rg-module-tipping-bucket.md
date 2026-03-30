# Rain Gauge Module — DIY Tipping Bucket
**Status:** Pre-build reference (outsourced model, design in progress)
**Node:** Node A only
**Interface:** GX16-4P → PORT-RG on core board

---

## Overview

The Rain Gauge Module measures rainfall via a DIY tipping bucket mechanism. Each tip of the seesaw bucket is detected by a KY-003 hall effect sensor with a magnet on the pivot arm. Tip count is converted to rainfall volume at the base station.

The tipping bucket housing is a separate modular cylinder mounted lower on the pole. Outsourced model — fabrication details tracked separately.

---

## Mechanism

| Parameter | Value |
|---|---|
| Seesaw length | 80 mm total (40 mm each side) |
| Bucket dimensions | 30 × 20 × 15 mm each |
| Funnel opening | ⌀100–120 mm |
| Funnel neck | ⌀20–25 mm |
| Volume per tip (0.2mm calibration, ⌀110mm funnel) | ~3.7 ml |
| Detection | KY-003 hall sensor + magnet on pivot arm |
| Housing | Cylindrical, ~150 × 150 × 100 mm outer |
| Bottom drain | ⌀8 mm hole |
| Top screen | Mesh to catch debris |

---

## Sensor: KY-003 Hall Effect

| Parameter | Value |
|---|---|
| Supply voltage | 3.3V or 5V |
| Output | Digital (active LOW on magnet detect) |
| PCB size | 15 × 10 mm |
| Header | 3-pin 2.54mm (VCC, GND, OUT) |

---

## Calibration

Each tip = one rainfall increment.
At ⌀110mm funnel: 1 tip ≈ 0.2mm rainfall = 3.7ml collected.
Calibration adjustable in firmware by changing `ML_PER_TIP` and `FUNNEL_AREA_CM2` constants.

---

## Board Interface (PORT-RG)

| Pin | Signal | Notes |
|---|---|---|
| 1 | VCC | 3.3V |
| 2 | GND | |
| 3 | SIGNAL | KY-003 OUT → ESP32 interrupt pin (FALLING edge) |
| 4 | DETECT | Module pulls to GND; HIGH = absent |

---

## Mounting

- Separate housing (outsourced design), mounted lower on pole below core enclosure
- GX16-4P cable port — placement on housing TBD (decide during housing design)
- Cable runs up to PORT-RG on core enclosure with drip loop
- Pole mount arm: PVC pipe (off-the-shelf) + 3D printed clamp joint (send to print service)

---

## Open Items

- [ ] Finalize outsourced housing model dimensions
- [ ] Confirm magnet-to-sensor gap (5–10mm max sensing distance for KY-003)
- [ ] Calibration test: tip count vs measured water volume
- [ ] Funnel screen mesh size (debris vs. fine rain droplets)
