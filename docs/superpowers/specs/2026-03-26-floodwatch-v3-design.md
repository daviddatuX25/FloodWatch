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

Clog (rising water + dropping flow rate) is a maintenance issue, not a flood alert. It uses a distinct alert category:
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

## Updated BOM Summary

| Item | v2 | v3 | v3 + connectors |
|------|----|----|-----------------|
| Node A | ₱1,890 | ₱1,940 | ₱1,970 |
| Node B | ₱1,351 | ₱1,401 | ₱1,431 |
| Base station | ₱200 | ₱200 | ₱200 |
| Shared | ₱322 | ₱322 | ₱322 |
| **Total** | **₱3,763** | **₱3,863** | **₱3,923** |

Connector additions: JST-XH connector sets (₱20) + DIP switch (₱10) = ₱30 per node.
