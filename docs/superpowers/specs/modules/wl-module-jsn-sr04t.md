# Water Level Module — JSN-SR04T
**Status:** Pre-build reference (component selected, not yet assembled)
**Node:** All field nodes
**Interface:** GX16-5P → PORT-WL on core board

---

## Overview

The Water Level Module measures water surface distance using a waterproof ultrasonic sensor (JSN-SR04T). Distance readings are used for rate-of-rise calculation — the primary early warning indicator.

---

## Sensor: JSN-SR04T Waterproof Ultrasonic

| Parameter | Value |
|---|---|
| Interface | Trigger/Echo (T mode) |
| Range | 20–600 cm |
| IP rating | IP67 |
| Supply voltage | 5V |
| Body OD | 22 mm |
| Thread | M16 × 1 (standard pipe thread) |
| Cable length | ~2.5 m, 4-wire |
| Beam cone angle | ~23° |
| Min clear radius at water surface | ~30 cm (at 1m height) |

---

## Mounting Spec

| Parameter | Value |
|---|---|
| Pole arm | PVC pipe (off-the-shelf), extends arm over canal |
| Arm joint | 3D printed clamp (sent to print service) — attaches PVC arm to pole |
| Sensor holder | Standard PVC 90° elbow fitting, sensor threads in via M16 thread |
| Aim | Straight down at water surface |
| Height above normal water | As high as practical; min 20cm clearance |
| Beam clearance | Keep 30cm radius at water surface free of structure |
| No separate enclosure | JSN-SR04T is IP67; cable runs directly to core enclosure |
| Cable entry to core | Via cable gland on core enclosure (no junction box needed) |
| Cable drip loop | Required — loop down before entering core enclosure |
| GX16-5P | On core enclosure panel only (no submodule box) |

---

## Board Interface (PORT-WL)

| Pin | Signal | Notes |
|---|---|---|
| 1 | VCC | 5V |
| 2 | GND | |
| 3 | TRIG | ESP32 GPIO output → sensor (10µs pulse) |
| 4 | ECHO | Sensor → ESP32 GPIO input — **5V output, needs divider** |
| 5 | DETECT | Module pulls to GND; HIGH = absent |

> **⚠ Level shift required on ECHO pin:** JSN-SR04T ECHO outputs 5V logic. ESP32 GPIO is not 5V tolerant.
> Use a simple resistor divider on the ECHO line before the ESP32 pin:
> ```
> ECHO (5V) ── 1kΩ ──┬── ESP32 GPIO
>                    │
>                   2kΩ
>                    │
>                   GND
> ```
> This divides 5V → 3.3V. Add this to the core PCB, not the module cable.

---

## 3D Print Scope (send to print service)

1. PVC pipe-to-pole clamp/joint

> PVC 90° elbow for sensor hold is off-the-shelf hardware store. Only the pole joint needs printing.

## Open Items

- [x] Confirm trigger/echo mode — confirmed, cart version is T mode
- [x] 1kΩ + 2kΩ resistors covered by resistor kit in cart
- [ ] Finalize PVC arm length for target deployment height
- [ ] Verify M16 thread fits standard PVC 90° elbow (hardware store check)
