---
title: FloodWatch v3 Design
date: 2026-03-26
status: approved
---

# FloodWatch v3 — Design Spec

## Context

This spec captures design decisions made during brainstorming that extend the v2 proposal. Changes are driven by three questions raised by the team:
1. Should nodes have a physical sound alarm/actuator?
2. How do we prevent false Red alerts on canal deployments?
3. Is canal blockage detection a distinct use case from flood warning?

## Key Design Decisions

### 1. Dual-Pipeline Alert Model

The base station owns all alert decisions. Two pipelines run independently:

- **Field pipeline** — LoRa downlink → buzzer + LED on each node. Targets maintenance crews on-site. Helps them locate which node is in alarm state.
- **Official pipeline** — SMS (AT+CMGS) + Firebase push → dashboard. Targets DRRMO, barangay captain, tanod, maintenance crew.

Nodes are dumb actuators: they report sensor data up and execute commands down. They never self-trigger.

### 2. Revised Alert Thresholds

Red now requires **both** conditions:
- Rate-of-rise > 30cm in 10 minutes
- Absolute water level ≥ MDRRMO official evacuation height

This dual-condition gate eliminates false Red alerts from rapid-but-safe water level changes.

### 3. Clog Detection as Separate Category

Clog detection triggers when water level (JSN-SR04T) rises while surface velocity (HB100) drops — indicating upstream blockage rather than rainfall-driven rise. It is a maintenance issue, not a flood alert. It uses a distinct alert category:
- Different buzzer pattern (slow triple beep) and LED color (blue)
- SMS to maintenance immediately
- Escalates to DRRMO if unresolved within configurable window
- Does NOT trigger Yellow/Orange/Red flood pipeline

### 4. Actuator Hardware per Node

Each node adds:
- Active buzzer module (5V, driven via 2N2222A) — ₱25
- 5mm LEDs (red, yellow, blue) with current-limiting resistors — ₱25
- Total addition per node: ~₱50

Power impact negligible — actuators only draw during alert states.

### 5. Buzzer Patterns

| State | Pattern | LED |
|-------|---------|-----|
| Yellow | Short single beep (repeating) | Yellow |
| Orange | Fast double beep (repeating) | Orange (R+Y) |
| Red | Continuous tone | Red |
| Clog | Slow triple beep (repeating) | Blue |
| Normal | Silent | Off |

### 6. Deployment Tiers

| Tier | Scope | Access |
|------|-------|--------|
| Local (prototype) | Single station, Orange Pi | On-site, no internet required |
| Cloud (future) | Multi-station | MDRRMO + public if approved |

### 7. Pre-Hardware Simulation

Firmware is validated in Wokwi (free ESP32 simulator) before uploading to physical boards. Wokwi natively supports ESP32 GPIO, LEDs, buzzers, ADC, Serial Monitor, and deep sleep. Components without Wokwi support (JSN-SR04T UART, HB100 microwave Doppler radar, KY-003, SX1278 LoRa) are stubbed — pushbuttons proxy interrupt-based sensors, hardcoded values proxy UART sensors. LoRaMesher mesh routing cannot be simulated and is validated hardware-only.

## Updated BOM Summary

| Item | v2 | v3 | v3 + connectors | current (actual cart) |
|------|----|----|-----------------|----------------------|
| Node A | ₱1,890 | ₱1,940 | ₱1,960 | **₱2,456** |
| Node B | ₱1,351 | ₱1,401 | ₱1,421 | **₱1,478** |
| Base station | ₱200 | ₱200 | ₱200 | ₱200 |
| Shared | ₱322 | ₱322 | ₱322 | ₱322 |
| **Total** | **₱3,763** | **₱3,863** | **₱3,903** | **₱4,456** |

Current column additions over v3+connectors: GX16 aviation connectors (5P, 4P×3, 2P) replacing cable glands, EMI copper foil tape for HB100 horn metallization, 1N5819 Schottky diode, PCB update. Enclosures (~₱488) and 3D print fees are deferred separately.
