# FloodWatch
## IoT-Based Flash Flood Early Warning System

**Submitted to:** Mr. Zeus Mendoza
**Subject:** Internet of Things
**Members:** David Datu Sarmiento, Brian Allan Jornales, Christine Lopez, Allyza Kaye Wong, Peter John Haboc

---

## I. Introduction

FloodWatch is an IoT system for real-time flash flood detection and early warning along canals and waterways. It deploys a scalable network of sensor nodes that communicate via LoRa radio in a multi-hop relay chain, passing aggregated data to a base station that handles machine learning inference, cloud sync, and SMS alerting.

The prototype demonstrates a minimum two-node deployment. All field nodes share identical hardware — role and capability are defined by attached sensors and position in the network, not by the board itself.

---

## II. Problem Statement

Flash floods in Philippine communities cause significant loss of life and property, particularly along canals and low-lying waterways. Automated early warning systems are rare, and those that exist typically require expensive cellular infrastructure at every sensor point. Communities near waterways often receive no timely automated warning before water levels become dangerous.

FloodWatch addresses this by distributing low-cost sensor nodes across a waterway, relaying data hop-by-hop to a single internet-connected base station — concentrating connectivity cost at one point while retaining wide sensing coverage.

---

## III. Objectives

1. Deploy a scalable multi-hop LoRa sensor network for real-time water level, flow rate, and rainfall monitoring.
2. Implement dynamic mesh routing via LoRaMesher so nodes can relay data without fixed infrastructure.
3. Demonstrate hardware modularity — any node can be upgraded with additional sensors without changing the base board or firmware.
4. Integrate Edge Impulse machine learning at the base station for anomalous surge classification.
5. Deliver alerts via native AT command SMS and a web dashboard with 3D canal water level simulation.
6. Power field nodes via solar and LiPo battery with barangay-maintained backup for shaded deployments.

---

## IV. System Architecture

### Network Topology

```
+------------------+        +------------------+        +------------------+
|   NODE A         |        |   NODE B         |        |   BASE STATION   |
|  (full config)   |        |  (minimal)       |        |                  |
|  Lolin32         |        |  Lolin32         |        |  Orange Pi Zero 3|
|  Ra-02 LoRa      |        |  Ra-02 LoRa      | WiFi   |  Edge Impulse ML |
|  JSN-SR04T       +------->|  JSN-SR04T       +------->|  Firebase        |
|  YF-S201 flow    |        |  relay + forward |        |  SMS alerts      |
|  KY-003 rain     |        |  battery only    |        |  Web dashboard   |
|  solar + battery |        |                  |        |  indoors         |
+------------------+        +--------^---------+        +------------------+
                                     |
                        dynamic LoRa meshing
                         (via LoRaMesher lib)
                                     |
                +--------------------+--------------------+
                |                                         |
+------------------+                        +------------------+
|   NODE C         |                        |   NODE D...      |
|  (any config)    |<---------------------->|  (any config)    |
|  Lolin32         |                        |  Lolin32         |
|  Ra-02 LoRa      |                        |  Ra-02 LoRa      |
|  sensors TBD     |                        |  sensors TBD     |
|  modular         |                        |  modular         |
+------------------+                        +------------------+

All field nodes run identical hardware.
Gateway role = whichever node is nearest to base station.
Nodes discover routes automatically — no fixed infrastructure needed.
```

### Data Flow (every 30 seconds)

```
  Field Nodes          Base Station          Firebase           Dashboard / SMS
  ───────────          ────────────          ────────           ───────────────
       │                    │                    │                    │
       │──LoRa packet──────►│                    │                    │
       │  (level/flow/rain) │                    │                    │
       │                    │──write readings───►│                    │
       │                    │                    │──live push────────►│
       │                    │                    │                    │
       │             ┌──────┴──────┐             │                    │
       │             │ Edge Impulse│             │                    │
       │             │ ML inference│             │                    │
       │             └──────┬──────┘             │                    │
       │                    │                    │                    │
       │             ┌──────┴─── surge? ─────────────────────────────┤
       │             │ YES        │                    │              │
       │             │           │──write alert───────►│              │
       │             │           │                    │──alarm UI────►│
       │             │           │──AT+CMGS──────────────────────────►│ SMS sent
       │             └──── NO: no action              │              │
       │                    │                    │                    │
  (repeat)            (repeat)              (persist)           (realtime)
```

### Node Configuration Summary

| Role | Board | Sensors | Power | Notes |
|------|-------|---------|-------|-------|
| Full node (A) | Lolin32 + Ra-02 | JSN-SR04T, YF-S201, KY-003 | Solar + LiPo | Primary sensing node |
| Minimal node (B) | Lolin32 + Ra-02 | JSN-SR04T | LiPo only | Relay + level reading; shaded sites |
| Extended (C, D…) | Lolin32 + Ra-02 | Any subset | Solar or battery | Plug into chain; same firmware |
| Base station | Orange Pi Zero 3 | — | Mains indoor | ML, Firebase, SMS control |

All field nodes run identical firmware. The gateway role is assigned to whichever node is closest to the base station — no special hardware required for this role.

---

## V. Routing Approach

Field nodes use the LoRaMesher library running on the ESP32 to handle dynamic multi-hop routing. Each node maintains a routing table and can forward packets from any other node. Packet IDs prevent duplicate retransmission. This is a relay chain topology for the prototype but scales to a partial mesh as more nodes are added. All nodes transmit at minimum 30-second intervals to comply with 433MHz duty cycle limits.

---

## VI. Hardware

### Wemos Lolin32 v1 — all field nodes
ESP-WROOM-32 with onboard LiPo charging circuit. Required for LoRaMesher. Deep sleep (~10µA idle) maximizes battery life between readings.

### SX1278 Ra-02 LoRa 433MHz
Better obstacle penetration than higher bands. Hop range 1–3km in semi-open terrain. Paired with 12dBi antenna via SMA-uFL adapter.

### JSN-SR04T Waterproof Ultrasonic Sensor
IP67-rated, 20–600cm range, UART mode. Distance-based output enables rate-of-rise calculation — the primary early warning indicator.

### YF-S201 Flow Sensor
1–30L/min hall-effect inline meter. Drop in flow rate alongside rising water level indicates blockage, not rainfall.

### KY-003 Hall Sensor — DIY Tipping Bucket Rain Gauge
Magnet on DIY tipping bucket. Each tip counted as rainfall volume increment. Provides predictive context and ML training data.

### 2N2222A NPN Transistor
Cuts Ra-02 power during ESP32 deep sleep, eliminating 1–2mA idle drain between transmissions.

### Orange Pi Zero 3 — Base Station
Full Linux SBC. Runs Edge Impulse Linux SDK, Firebase SDK, SMS triggering. Chosen for persistent process support and Python/Node.js environment.

### Active Buzzer Module + 5mm LEDs — all field nodes
Each node includes a 5V active buzzer and LED indicators (red, yellow, blue). Alert state and buzzer pattern are commanded via LoRa downlink from the base station — nodes do not self-trigger. Buzzer is switched via 2N2222A to eliminate idle drain during deep sleep.

---

## VII. Sensor Modularity

All nodes expose the same UART, SPI, and GPIO interfaces. Node B carries JSN-SR04T only. Flow sensor, rain gauge, and solar panel can be added without board changes or firmware reflash.

**Three demonstrated configurations:**
- Full node: water level + flow rate + rainfall + solar
- Standard node: water level + solar
- Minimal node: water level + battery only

---

## VIII. Edge Impulse ML Integration

### Rationale
Threshold alerting cannot distinguish gradual rainfall from sudden surge or adapt to seasonal baseline shifts.

### Model design
Rolling window of water level readings from all nodes → binary classification: normal flow or anomalous surge.

### Deployment
Inference on Orange Pi Zero 3 via Edge Impulse Linux SDK.

### Phased activation
Launches threshold-only. ML activates after 2–4 weeks of collected and labeled data. ML flags cross-check with thresholds before any alert is issued.

---

## IX. Alert Logic

### Flood Alert Levels

| Level | Trigger | Field Signal | Official Channel |
|-------|---------|-------------|-----------------|
| Yellow | Rise > 5cm in 10 minutes | Short beep, yellow LED | SMS to barangay tanod and DRRMO officer |
| Orange | Rise > 15cm in 10 minutes | Fast double beep, orange LED | SMS to barangay captain and resident list |
| Red | Rise > 30cm in 10 minutes **AND** water ≥ MDRRMO evacuation height | Continuous alarm, red LED | SMS all contacts + dashboard alarm |

Red requires both conditions. The MDRRMO absolute height gate prevents false Red alerts from rapid but still-safe water level changes (e.g., after a heavy rain that drains quickly).

### Canal Clog Detection

Detected when: water level is rising while flow rate is dropping — indicates blockage upstream, not rainfall-driven rise.

| Level | Trigger | Field Signal | Official Channel |
|-------|---------|-------------|-----------------|
| Clog | Level rising + flow rate dropping (Node A only) | Slow triple beep, blue LED | SMS to barangay maintenance immediately; escalate to DRRMO if unresolved within configurable window |

Clog alerts are intentionally separate from flood alert levels — they indicate a maintenance issue, not an imminent evacuation threat. A clog that goes uncleared can escalate into a flood alert independently.

---

## X. Software Stack

| Layer       | Tool                             | Cost       |
|-------------|----------------------------------|------------|
| Firmware    | Arduino IDE + LoRaMesher         | Free       |
| ML model    | Edge Impulse (academic tier)     | Free       |
| Backend     | Firebase Realtime Database       | Free tier  |
| Dashboard   | Three.js + Chart.js              | Free       |
| Hosting     | Firebase Hosting or Vercel       | Free tier  |
| SMS alerts  | A7680C AT+CMGS (future modular)  | Free       |
| Data SIM    | Prepaid load                     | ~₱50/month |

---

## XI. Bill of Materials and Costing

### Node A — full configuration

| Component                       | Price      |
|---------------------------------|------------|
| Wemos Lolin32 v1                | ₱192       |
| 18650 2200mAh cell              | ₱165       |
| 18650 holder 4-slot             | ₱37        |
| SX1278 Ra-02 LoRa 433MHz        | ₱305       |
| 12dBi antenna + SMA-uFL adapter | ₱139       |
| JSN-SR04T ultrasonic sensor     | ₱174       |
| YF-S201 flow sensor             | ₱135       |
| KY-003 hall sensor              | ₱20        |
| 6V 1W solar panel               | ₱249       |
| 2N2222A transistor              | ₱7         |
| Dupont MF cables                | ₱37        |
| PCB 5x7 universal board         | ₱55        |
| Enclosure                       | ₱175       |
| Cable glands                    | ₱30        |
| JSN-SR04T mounting bracket      | ₱35        |
| YF-S201 pipe fittings/tubing    | ₱75        |
| Tipping bucket parts (DIY)      | ₱55        |
| Active buzzer module (5V)       | ₱25        |
| 5mm LED — red, yellow, blue     | ₱20        |
| Current-limiting resistors      | ₱5         |
| JST-XH connector sets (4 ports) | ₱20        |
| 4-position DIP switch           | ₱10        |
| **Node A total**                | **₱1,970** |

### Node B — minimal configuration

| Component                       | Price      |
|---------------------------------|------------|
| Wemos Lolin32 v1                | ₱192       |
| 18650 2200mAh cell              | ₱165       |
| 18650 holder 4-slot             | ₱37        |
| SX1278 Ra-02 LoRa 433MHz        | ₱305       |
| 12dBi antenna + SMA-uFL adapter | ₱139       |
| JSN-SR04T ultrasonic sensor     | ₱174       |
| 2N2222A transistor              | ₱7         |
| Dupont MF cables                | ₱37        |
| PCB 5x7 universal board         | ₱55        |
| Enclosure                       | ₱175       |
| Cable glands                    | ₱30        |
| JSN-SR04T mounting bracket      | ₱35        |
| Active buzzer module (5V)       | ₱25        |
| 5mm LED — red, yellow, blue     | ₱20        |
| Current-limiting resistors      | ₱5         |
| JST-XH connector sets (4 ports) | ₱20        |
| 4-position DIP switch           | ₱10        |
| **Node B total**                | **₱1,431** |

### Base Station

| Component        | Price      |
|------------------|------------|
| Orange Pi Zero 3 | ₱0 (owned) |
| Enclosure        | ₱200       |
| **Total**        | **₱200**   |

### Shared One-Time

| Component                   | Price    |
|-----------------------------|----------|
| 2N2222A 5pcs pack           | ₱35      |
| Extra dupont bundle         | ₱37      |
| Breadboard 400pt (dev only) | ₱50      |
| Shipping estimate           | ₱200     |
| **Total**                   | **₱322** |

### Grand Total

| Item         | Cost       |
|--------------|------------|
| Node A       | ₱1,970     |
| Node B       | ₱1,431     |
| Base station | ₱200       |
| Shared       | ₱322       |
| **Total**    | **₱3,923** |

> Enclosure, bracket, fitting, and tipping bucket costs are mid-range estimates.

---

## XII. Power Design

~15mA average draw at 30s intervals yields ~146 hours standalone on 2200mAh. Solar extends this indefinitely. Node B is battery-only, demonstrating shaded deployment viability. 2N2222A eliminates Ra-02 idle drain during sleep.

Actuator power impact is negligible in practice. The buzzer (~30mA) and LEDs (~15mA combined) only draw during active alert states — which are rare events. Average power budget is effectively unchanged. The buzzer is also switched via 2N2222A and can be cut during deep sleep.

---

## XIII. Risk Assessment

| Risk | Mitigation |
|------|-----------|
| LoRa signal blockage | Site survey; 433MHz penetrates obstacles better than higher bands |
| Solar shading | Node B battery-only; spare cells maintained by barangay |
| Node submersion | Mount above flood height; IP65 enclosure |
| WiFi unavailability | A7680C 4G modular fallback |
| LoRaMesher instability | Pre-deployment routing tests; linear relay fallback |
| ML false positives | Cross-check with thresholds before issuing alerts |
| Insufficient ML data | Threshold-only launch; ML activates after 2–4 weeks |

---

## XIV. Scope and Limitations

Validates complete pipeline across two nodes, including dual-pipeline alert architecture, actuator control via LoRa downlink, and clog detection. Each additional node costs ~₱886 bare minimum (includes buzzer + LED). Full waterway coverage requires a formal site survey to determine node spacing and count.

### Deployment Tiers

| Tier | Location | Access | Scope |
|------|----------|--------|-------|
| Local | Orange Pi base station | On-site (no internet required) | Single station, field team |
| Cloud | Firebase + hosted dashboard | MDRRMO, barangay officials, public (if approved) | Multi-station view, all base stations |

The prototype implements the local tier. Cloud multi-station view is future scope.

---

## XV. Future Enhancements

- A7680C 4G LTE modular integration (base station internet fallback)
- Additional upstream nodes
- Edge Impulse retraining pipeline
- PAGASA forecast API integration
- Mobile companion app on Firebase backend
- Cloud multi-station dashboard (requires MDRRMO approval for public access)
- Public-facing web view per canal for residents (pending MDRRMO authorization)

---

*Submitted in partial fulfillment of the requirements for Internet of Things*