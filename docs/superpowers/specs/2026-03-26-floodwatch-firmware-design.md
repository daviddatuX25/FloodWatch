---
title: FloodWatch Firmware Design
date: 2026-03-26
status: approved
scope: Sub-project 1 of 4 — Field Node Firmware
---

# FloodWatch Firmware — Design Spec

## Context

This spec covers the firmware for all field sensor nodes in the FloodWatch system. It is the first of four sub-projects (Firmware → Base Station → Dashboard → ML Pipeline). The firmware must be completed first because all other subsystems depend on it producing data.

The team (5 members) has basic Arduino tinkering experience but no prior LoRa, SPI, or deep sleep work. Hardware has not been purchased yet. Target completion: ~1 month, with a 2-month buffer. Maximum 2 parallel work streams.

## Build Approach

Layer-by-layer. Each layer is a milestone completed and verified before the next begins. Each layer produces a testable, demonstrable result.

### Layer 1 — Individual Sensor Tinkering (parallel-friendly)

Each sensor gets its own standalone sketch on a single Lolin32. Goal: understand how it works, get reliable readings, and figure out physical mounting.

| Sensor | What to prove | Physical tinkering |
|---|---|---|
| JSN-SR04T (water level) | UART mode reads, stable distance at 20–600cm range | Mounting bracket angle, cable gland entry, IP67 seal test |
| HB100 (Doppler radar) | ADC sampling of amplified IF output, FFT frequency extraction, velocity calculation | Mounting angle 30-45° over canal, LM358 op-amp circuit assembly, noise floor baseline |
| KY-003 + tipping bucket (rain) | Magnet trigger counting, tip-to-mm calibration | DIY bucket mechanism, funnel, pivot, magnet placement |
| Buzzer + LEDs (actuators) | Beep patterns (yellow/orange/red/clog), LED switching | Panel-mount LEDs in enclosure lid, buzzer positioning for audibility |

Two groups can work in parallel here (e.g., Group 1: water level + flow, Group 2: rain gauge + actuators).

### Layer 2 — Point-to-Point LoRa

Two Lolin32s, each with an Ra-02 module. Raw SPI. One sends, one receives. Proves wiring, antenna connection, and basic packet exchange.

### Layer 3 — LoRaMesher Multi-Hop

Replace raw LoRa with LoRaMesher library. Two nodes + gateway role. Bi-directional: sensor data up, commands down. Packet format defined and validated here.

### Layer 4 — Compose: Sensors + Mesh + Actuators

Wire sensors from Layer 1 onto the meshed nodes from Layer 3. Base station sends alert commands, nodes respond with buzzer/LED patterns.

### Layer 5 — Power Management

2N2222A transistor switching for Ra-02 and buzzer during deep sleep. Sleep/wake cycle at 30s intervals. Solar charging validation on Lolin32's onboard circuit.

### Layer 6 — Enclosure & Physical Integration

Final node assembly: PCB soldering (off breadboard), enclosure layout, cable glands, sensor mounts, weatherproofing, aesthetics.

---

## Packet Protocol

### Uplink (node → base station, every 30 seconds)

| Field | Size | Description |
|---|---|---|
| node_id | 1 byte | Unique per node (set via DIP switch) |
| packet_type | 1 byte | 0x01 = sensor data, 0x02 = heartbeat |
| water_level_cm | 2 bytes | JSN-SR04T reading (unsigned, cm) |
| surface_velocity | 2 bytes | HB100 reading (mm/s, 0 if no sensor) |
| rain_tips | 1 byte | Tip count since last packet (0 if no gauge) |
| battery_mv | 2 bytes | Battery voltage in millivolts |
| flags | 1 byte | Bit flags: has_flow (HB100 velocity sensor present), has_rain, low_battery |

Total: 10 bytes.

Nodes without flow or rain sensors send 0 in those fields with the corresponding flag unset. The base station uses flags to know which fields to interpret.

### Downlink (base station → node, via LoRaMesher)

| Field | Size | Description |
|---|---|---|
| target_node_id | 1 byte | 0xFF = broadcast to all |
| alert_state | 1 byte | 0=normal, 1=yellow, 2=orange, 3=red, 4=clog |

Total: 2 bytes.

Nodes map `alert_state` directly to a buzzer/LED pattern. They do not interpret alert levels — the base station owns all alert logic.

### Buzzer/LED Pattern Mapping

| alert_state | Buzzer Pattern | LED |
|---|---|---|
| 0 (normal) | Silent | All off |
| 1 (yellow) | Short single beep, repeating | Yellow |
| 2 (orange) | Fast double beep, repeating | Red + Yellow |
| 3 (red) | Continuous tone | Red |
| 4 (clog) | Slow triple beep, repeating | Blue |

---

## Pin Assignments

Standard pin map for all Lolin32 field nodes. Unused sensors simply aren't wired.

### Ra-02 LoRa (SPI)

| Ra-02 Pin | Lolin32 Pin | Notes |
|---|---|---|
| SCK | GPIO 18 | SPI clock |
| MISO | GPIO 19 | SPI data in |
| MOSI | GPIO 23 | SPI data out |
| NSS | GPIO 5 | Chip select |
| RST | GPIO 14 | LoRa reset |
| DIO0 | GPIO 26 | Interrupt (packet ready) |

### Ra-02 Power Switching (2N2222A)

| Component | Lolin32 Pin | Notes |
|---|---|---|
| 2N2222A base (via 1kΩ resistor) | GPIO 4 | HIGH = Ra-02 powered on |
| 2N2222A collector | Ra-02 VCC | Switches 3.3V to module |

### Sensors

| Sensor | Lolin32 Pins | Notes |
|---|---|---|
| JSN-SR04T (UART mode) | TX→GPIO 16, RX→GPIO 17 | Hardware Serial2 |
| HB100 IF output (amplified) | Signal→GPIO 27 | ADC-capable pin (via LM358) |
| KY-003 (rain tip) | Signal→GPIO 25 | Interrupt-capable pin |

### Actuators

| Component | Lolin32 Pin | Notes |
|---|---|---|
| Buzzer (via 2N2222A) | GPIO 32 | Transistor-switched |
| LED Red | GPIO 33 | Via 220Ω resistor |
| LED Yellow | GPIO 12 | Via 220Ω resistor |
| LED Blue | GPIO 13 | Via 220Ω resistor |

### DIP Switch (Node ID)

| Bit | Lolin32 Pin | Notes |
|---|---|---|
| Bit 0 (LSB) | GPIO 21 | With external 10kΩ pull-up |
| Bit 1 | GPIO 22 | With external 10kΩ pull-up |
| Bit 2 | GPIO 34 | Input-only, external 10kΩ pull-up |
| Bit 3 (MSB) | GPIO 36 | Input-only, external 10kΩ pull-up |

### Battery Monitoring

| Component | Lolin32 Pin | Notes |
|---|---|---|
| Battery voltage divider | GPIO 35 (ADC) | Two resistors, reads LiPo voltage |

Pin choices avoid GPIO 0, 2, 15 (boot-sensitive) and keep SPI on the default bus. Several GPIOs remain free for future use.

---

## Firmware Architecture

Single codebase for all nodes. Behavior determined at runtime by auto-detecting which sensors are plugged in.

### Auto-Detection at Boot

| Sensor | Detection Method |
|---|---|
| JSN-SR04T | Send UART ping on Serial2, check for echo response within timeout |
| HB100 | Read ADC on GPIO 27 for 100ms — if signal variance exceeds noise threshold, HB100 amplifier circuit is present and active |
| KY-003 (rain) | Pull-up on GPIO 25 — same detection pattern as before |
| Ra-02 LoRa | SPI read of version register (0x12 should return 0x12 for SX1278) |
| Buzzer/LEDs | Always present — no detection needed |

A `detectHardware()` function in `setup()` populates:

```cpp
struct NodeConfig {
  uint8_t node_id;       // read from DIP switch
  bool has_flow;
  bool has_rain;
  bool has_lora;
  bool has_solar;        // battery voltage > charge threshold = solar present
};
```

No reflash needed to change node configuration. Plug a sensor in, reboot, it's detected.

**Node ID:** 4-position DIP switch (~₱10) on the PCB. Set node address physically. Supports up to 15 nodes.

### Firmware Loop

```
setup()
  → read DIP switch → node_id
  → detectHardware() → NodeConfig
  → init detected sensors
  → init LoRaMesher
  → init actuators

loop()
  → wake from deep sleep
  → power on Ra-02 (GPIO 4 HIGH)
  → wait 10ms (module boot)
  → read sensors (~200ms) → build uplink packet
  → LoRaMesher: send packet (~500ms)
  → LoRaMesher: listen for downlink (~1s window)
  → if command received → set buzzer/LED pattern
  → power off Ra-02 (GPIO 4 LOW)
  → if alert_state == normal → enter deep sleep (30s)
  → if alert_state != normal → stay awake, run buzzer pattern, keep listening
```

### File Structure

| File | Responsibility |
|---|---|
| `FloodWatch_Node.ino` | Main setup/loop, sleep/wake cycle |
| `config.h` | Pin assignments, constants |
| `detect.h/.cpp` | Auto-detect plugged sensors, read DIP switch, populate NodeConfig |
| `sensors.h/.cpp` | Read JSN-SR04T, HB100 (ADC + FFT), KY-003, battery ADC (skips absent sensors) |
| `lora_comm.h/.cpp` | LoRaMesher init, send uplink, receive downlink |
| `actuators.h/.cpp` | Buzzer patterns, LED states, map alert_state to output |
| `power.h/.cpp` | Transistor switching, deep sleep entry/exit |

Each file maps to a build layer. Team members working on Layer 1 only touch `sensors.h`. Layer 2–3 work only touches `lora_comm.h`. No one needs to understand the whole codebase to work on their piece.

---

## Physical Modularity — Connectors

All sensor and power connections use JST-XH connectors. No soldered sensor wires.

| Connector | Used For | Why JST-XH |
|---|---|---|
| JST-XH 4-pin | JSN-SR04T (VCC, GND, TX, RX) | Keyed, snaps in, can't mis-plug |
| JST-XH 3-pin | HB100 amplified signal (VCC, GND, Signal) | Same family |
| JST-XH 3-pin | KY-003 rain gauge (VCC, GND, Signal) | Same family |
| JST-XH 2-pin | Solar panel (V+, GND) | Matches Lolin32's JST battery style |
| Pin header 2x4 | Ra-02 LoRa module | Daughter-board with SMA connector, slides onto headers |

Benefits:
- **Setup:** Plug in sensors, set DIP switch, power on
- **Repair:** Unplug broken sensor, plug in replacement. No soldering
- **Upgrade:** Add flow sensor to a minimal node later — just plug it in
- **Testing:** During Layer 1, plug one sensor at a time into the same board

Additional BOM per node: JST-XH connector sets (~₱5 each × 4 = ₱20) + DIP switch (~₱10) = ~₱30.

---

## Power Management

### Sleep/Wake Cycle (30-second interval)

```
DEEP SLEEP (30s) → Wake
  → GPIO 4 HIGH (power on Ra-02)
  → wait 10ms (module boot)
  → read sensors (~200ms)
  → LoRaMesher: send uplink (~500ms)
  → LoRaMesher: listen for downlink (~1s window)
  → if alert command → run buzzer/LED pattern
  → GPIO 4 LOW (power off Ra-02)
  → GPIO 32 LOW (power off buzzer if not alerting)
  → DEEP SLEEP
```

### Active Alert Override

When a node is in an active alert state (buzzer sounding), it does NOT deep sleep. It stays awake running the buzzer pattern and keeps LoRa powered to receive updated commands (e.g., base station clears the alert). Returns to normal sleep cycle once alert_state goes back to 0.

### Power Budget

| State | Current | Duration | Notes |
|---|---|---|---|
| Deep sleep | ~10µA | ~28s | ESP32 ULP only |
| Active (sensors + LoRa) | ~120mA | ~2s | Burst per cycle |
| Average draw (no alert) | ~8mA | — | ~275 hours on 2200mAh |
| Alert state (buzzer + LEDs + LoRa) | ~170mA | Continuous until cleared | Solar or battery drain |

### Solar Charging

Lolin32's onboard TP4054 charges LiPo directly from the 6V 1W panel via the 5V pin. No extra charge controller needed. Solar panel connects via JST-XH 2-pin through cable gland.

### Battery Monitoring

Voltage divider on GPIO 35 reads LiPo voltage. Below 3.3V → `low_battery` flag set in uplink packet → base station notifies maintenance crew.

---

## Enclosure Design

IP65 ABS box, approximately 150×100×70mm.

### Exterior (Lid)

```
┌─────────────────────────────┐
│  ┌─────┐   ┌─────┐         │
│  │ LED │   │ LED │  ○ LED  │  Panel-mount LEDs (R, Y, B)
│  │ Red │   │ Yel │  Blue   │
│  └─────┘   └─────┘         │
│         ┌──────────┐        │
│         │  Buzzer   │       │  Behind mesh vent (sound out, water blocked)
│         │  (mesh)   │       │
│         └──────────┘        │
│  [DIP SW]                   │  Node ID, accessible from outside
└─────────────────────────────┘
```

### Interior

```
┌─────────────────────────────┐
│  ┌──────────┐  ┌─────────┐ │
│  │ Lolin32  │  │ Ra-02   │ │  Ra-02 on pin header daughter-board
│  │          │  │ + uFL   │ │
│  └──────────┘  └─────────┘ │
│  ┌────────┐                 │
│  │ 18650  │  JST-XH ports  │  Labeled: LEVEL, FLOW, RAIN, SOLAR
│  │ holder │  ○ ○ ○ ○       │
│  └────────┘                 │
└─────────────────────────────┘
```

### Cable Glands (bottom of box only)

- Antenna SMA feedthrough (side)
- JSN-SR04T cable
- HB100 signal cable (if present)
- KY-003 cable (if present)
- Solar panel cable (if present)

All glands on bottom — water drips down, not into the box.

### Mounting & Field Concerns

| Concern | Solution |
|---|---|
| Water ingress | Cable glands bottom-only; IP65 enclosure; mesh vent for buzzer |
| Sensor mounting | JSN-SR04T on bracket pointing down at water; HB100 mounted at 30-45° angle above canal surface, aimed along flow direction |
| Rain gauge | DIY tipping bucket on small mast/bracket above enclosure |
| Antenna | SMA feedthrough on side, 12dBi antenna mounted vertically above box |
| Repairability | Open lid → all JST-XH connectors accessible. Unplug, swap, re-plug |
| Battery swap | 18650 holder slot — pull cell, insert charged one. No tools needed |
| Site mounting | Zip tie or U-bolt loops on back of enclosure → mount to railing/post |

---

## Updated BOM Impact

Modular connector additions per node:

| Addition | Cost |
|---|---|
| JST-XH connector sets (4 ports) | ~₱20 |
| 4-position DIP switch | ~₱10 |
| **Per-node addition** | **~₱30** |

Updated node totals (from v3 design):

| Node | v3 Price | + Connectors | New Total |
|---|---|---|---|
| Node A (full) | ₱1,940 | +₱30 | ₱1,960 |
| Node B (minimal) | ₱1,401 | +₱30 | ₱1,431 |

Grand total: ₱1,960 + ₱1,431 + ₱200 (base station) + ₱322 (shared) = **₱3,913**

---

## Scope Boundaries

**In scope (this spec):**
- All 6 firmware build layers
- Physical sensor tinkering and mounting
- Enclosure design and weatherproofing
- Complete packet protocol (uplink + downlink)
- Auto-detection and modular connector design

**Out of scope (separate sub-projects):**
- Base station software (Orange Pi — receives packets, alert logic, SMS, Firebase)
- Web dashboard (Three.js, Chart.js, Firebase)
- ML pipeline (Edge Impulse training and deployment)

**Dependency this spec creates:** The base station sub-project depends on the packet format defined here (10-byte uplink, 2-byte downlink).
