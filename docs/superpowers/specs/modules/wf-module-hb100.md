# Water Flow Module — HB100 + Pyramidal Horn
**Status:** Pre-build reference (concept validated, not yet fabricated)
**Node:** Node A only
**Interface:** GX16-4P → PORT-WF on core board

---

## Overview

The Water Flow Module measures surface water velocity using a non-contact microwave Doppler radar (HB100, 10.525 GHz). A DIY pyramidal horn antenna narrows the HB100's native 80°×40° beam, rejecting moving objects on canal banks (pedestrians, vehicles) and focusing energy on the water surface.

The canal deployment context is narrow (~0.5m wide Philippine estero). Static canal banks return ~0 Doppler, so beam width primarily affects rejection of moving bank-side interference — not false velocity from the walls themselves.

---

## Sensor: HB100 Microwave Doppler Radar

| Parameter | Value |
|---|---|
| Frequency | 10.525 GHz (X-Band) |
| Wavelength (λ) | 28.5 mm |
| Antenna type | Patch antenna (linear polarization) |
| Native beam (H × V) | 80° × 40° |
| IF output | Analog, microvolt-level |
| Supply voltage | 5V |
| PCB dimensions | ~38 × 25 mm |
| Mounting holes | 2× M3, ~33mm spacing (long axis) — **verify on actual board** |

**Signal chain:** HB100 IF output → LM358 op-amp amplifier (2-stage) → ESP32 ADC → FFT → Doppler frequency → velocity

Velocity formula: `V = Fd / (2 × f₀/c × cos θ)`
Simplified at 10.525 GHz: `V (m/s) = Fd (Hz) / 72 × cos(35°)`

---

## Horn Antenna — Pyramidal Shroud

The horn acts as a directive shroud, not a classical waveguide-fed horn. The HB100 patch antenna radiates into the horn flare. Internal metallization reflects and focuses the beam.

### Dimensions

| Component | Dimension | Notes |
|---|---|---|
| Throat (a × b) | 45 × 37 mm | Fits snugly over HB100 PCB face |
| Aperture (A × B) | 118 × 90 mm | Mouth — narrower default for ~0.5m canal |
| Flare length (L) | 120 mm | Minimizes phase error and side lobes |
| Wall thickness | 2–3 mm | Structural; PETG/ASA minimum |
| Mounting flange | 4× M3 holes | Match HB100 PCB hole layout exactly |

### Expected Performance

| Parameter | Estimated Value |
|---|---|
| H-plane beamwidth (3dB) | ~25° |
| V-plane beamwidth (3dB) | ~35° |
| Estimated gain | +6 to +9 dBi over stock |
| Spot width at 2m, 35° angle | ~1.0–1.3 m (along canal axis) |

> **Note for 0.5m canal:** Doppler measurement is velocity-based, not position-based. Static canal banks contribute ~0 Doppler return. Horn primarily rejects moving objects on the bank. The ~25° horizontal beam is the default; further narrowing (larger aperture) is a future option if needed.

### Fabrication Requirements

- **Material:** PETG or ASA — required for outdoor UV/heat resistance. PLA not suitable.
- **Internal metallization:** All interior surfaces lined with smooth aluminum foil tape or conductive spray. No gaps, no wrinkles. Overlap seams by ≥5mm.
- **Throat seal:** Foam gasket (2–3mm) between HB100 PCB face and horn throat flange. Eliminates leakage and ghost reflections.
- **Radome (recommended):** 2mm flat PETG cover over aperture mouth. PETG is microwave-transparent at 10.525 GHz. Prevents debris and moisture ingress.
- **Print orientation:** Horn open-face down on printer bed for best layer adhesion along flare walls.

---

## Submodule Housing

The assembly splits into two parts:
1. **Horn** — 3D printed (sent to print service), lined with aluminum foil tape
2. **Electronics box** — off-the-shelf IP65 ABS enclosure (S size: 65×58×35mm), mounts to the back of the horn

Metal enclosure will block the signal — never use metal for the horn or electronics box.

| Parameter | Spec |
|---|---|
| Electronics enclosure | IP65 ABS, S size — **65×58×35mm** |
| Contents | HB100 PCB + LM358 sub-board (30×40mm PCB cut-down) |
| Cable entry | GX16-4P on **side face** |
| Horn attachment | S box bolts flush to horn throat rear face |
| Horn aperture | Open face forward, optional radome (2mm ABS/PETG flat cover) |

---

## Mounting Spec

| Parameter | Value |
|---|---|
| Deployment height | ~2m above water surface |
| Beam angle | 35° downward from horizontal |
| Aim direction | **Along canal flow** (upstream or downstream) — NOT across canal |
| Pole arm | PVC pipe (off-the-shelf), attached to pole |
| Bracket | 3D printed slotted-arc angle adjuster, single M4 pivot bolt |
| Cable drip loop | Required — 100mm downward loop before cable rises to enclosure |

**Why along-canal and not across:**
The canal is narrow. Aimed along the canal, the beam travels down the water corridor. Aimed across, the beam hits the far bank immediately. Surface water velocity vector has maximum projection along the canal axis, maximizing Doppler sensitivity.

---

## Board Interface (PORT-WF)

| Pin | Signal | Notes |
|---|---|---|
| 1 | VCC | 5V for HB100; LM358 can run on 3.3V or 5V |
| 2 | GND | Common ground |
| 3 | ANALOG OUT | Amplified IF signal → ESP32 ADC pin |
| 4 | DETECT | Module pulls to GND when connected; ESP32 INPUT_PULLUP reads LOW = present |

---

## Signal Conditioning (LM358 Amp Sub-board)

The HB100 IF output is microvolt-level and requires amplification before the ESP32 ADC.

**Reference circuit (2-stage non-inverting):**
- Stage 1: Gain ~100× (20 dB)
- Stage 2: Gain ~10× (20 dB)
- Total gain: ~1000× (60 dB)
- High-pass filter at input: ~10 Hz (rejects DC offset)
- Low-pass filter at output: ~500 Hz (Doppler range for water velocity 0–7 m/s)

**Target velocity range for canal flow:**
- Normal flow: 0.1–1.5 m/s → Doppler: 7–108 Hz
- Flood surge: 1.5–5 m/s → Doppler: 108–360 Hz
- Max useful range: ~7 m/s → ~504 Hz

The LM358 amp sub-board mounts inside the submodule housing alongside the HB100.

---

## Bill of Materials (WF Module Additions)

| Component | Est. Cost | Notes |
|---|---|---|
| HB100 Doppler radar module | ₱150 | Already in Node A BOM |
| LM358 DIP-8 op-amp | ₱15 | Already in Node A BOM |
| Resistors + capacitors (amp circuit) | ₱20 | Already in Node A BOM |
| Aluminum foil tape roll | ~₱60 | Shared across prints |
| IP65 ABS box S size (65×58×35mm) | ~₱80 | Electronics housing (HB100 + LM358) |
| GX16-4P connector set (male + female) | ~₱76 | Already in cart |
| LM358 sub-board PCB | — | Cut from 8×12cm universal board (already in cart) |
| M4 bolt + nut (bracket pivot) | ₱10 | |
| Foam gasket strip (throat seal) | ₱15 | |
| **Module total (add-on cost, excl. print service fee)** | **~₱340** | |

> **3D print scope (send to print service):**
> 1. Pyramidal horn body (PETG/ASA, ~100g)
> 2. Slotted-arc angle adjuster bracket
> 3. PVC pipe-to-pole joint clamp
>
> Everything else is off-the-shelf (PVC pipe arms, IP65 ABS S-box). Print service fee tracked separately once designs are ready.

---

## Open Items (Pre-Fabrication Checklist)

- [ ] Measure actual HB100 PCB mounting hole spacing (verify ~33mm before printing flange)
- [ ] Confirm canal width at deployment site — affects minimum horn aperture choice
- [ ] Decide: horn integrated into housing, or snap-fit modular attachment
- [ ] Validate LM358 amp gain stages against ADC input range on Lolin32 (3.3V max)
- [ ] Determine final mounting pole arm length for 2m height clearance above flood level
- [ ] Test horn beam pattern before potting (simple walk-test with oscilloscope on IF output)

---

## References

- HB100 datasheet: application note on IF output and recommended load
- Horn antenna sizing: 56λ/A rule for 3dB beamwidth estimate
- Doppler velocity formula: `V = Fd × λ / (2 × cos θ)` — standard CW radar
- Metallization guidance: pjrc.com forum thread on HB100 + FFT flow sensing
- MDPI Electronics 2025 — pyramidal horn DIY at X-band
