

Internet of Things — Final’s Project Proposal

# FloodWatch

IoT-Based Flash Flood Early Warning System  
with Edge ML Surge Detection

Submitted to: Mr. Zeus Mendoza  
Subject: Internet of Things

**Members:**  
David Datu Sarmiento  
Brian Allan Jornales  
Christine Lopez  
Allyza Kaye Wong  
Peter John Haboc

## 

## I. Introduction

FloodWatch is a proposed IoT system for real-time flash flood detection and early warning along canals and waterways in provincial and city settings. Three low-cost sensor nodes communicate via LoRa radio in a hop-relay topology, pushing aggregated data through a WiFi-connected gateway to a cloud backend. Alerts reach barangay officials and residents via SMS using native AT commands on the gateway module — no paid API is required.

The system incorporates Edge Impulse machine learning at the gateway node for on-device anomalous surge detection, elevating it from a basic sensor network to an intelligent edge IoT system.

## II. Problem Statement

Flash floods in Philippine communities frequently cause loss of life and property, particularly along canals. Automated warning systems are rare, and those that exist typically require expensive cellular modules at every sensing point — making wide coverage cost-prohibitive. FloodWatch addresses this by concentrating connectivity cost at a single gateway node while retaining distributed sensing across the waterway.

## III. Objectives

1. Deploy a 3-node LoRa mesh sensor network for real-time water level monitoring.  
2. Implement hop-based data relay to eliminate per-node SIM module cost.  
3. Integrate Edge Impulse on-device ML for anomalous surge classification at the gateway.  
4. Deliver alerts via native AT command SMS and a Three.js 3D web dashboard.  
5. Power all nodes sustainably via solar \+ LiPo battery with barangay-maintained backup.

## IV. System Architecture

Node 1 (upstream)        Node 2 (relay)           Node 3 (gateway)  
Wemos Lolin32 v1         Arduino Nano             Wemos Lolin32 v1  
Ra-02 LoRa 433MHz        Ra-02 LoRa 433MHz        Ra-02 LoRa 433MHz  
JSN-SR04T sensor         JSN-SR04T sensor         JSN-SR04T sensor  
Solar \+ LiPo             Solar \+ LiPo             Solar \+ LiPo  
     |                        |                   WiFi → Firebase  
     └──── LoRa hop ──────────┘──── LoRa hop ─── SMS via AT commands  
							   A7680C 4G (modular slot)

Node 3 is placed near a barangay hall with WiFi. If WiFi becomes unreliable, the A7680C 4G module plugs into the existing UART header — only a firmware compile flag needs to change.

## 

## V. Hardware Selection and Rationale

### Wemos Lolin32 v1 — Nodes 1 and 3

Based on the ESP-WROOM-32, the Lolin32 includes an onboard LiPo charging circuit, eliminating a separate charge module on these two nodes. Node 1 uses ESP32 deep sleep (\~10µA idle) to maximize battery life between readings. Node 3 uses its built-in WiFi for cloud connectivity.

### Arduino Nano — Node 2

Node 2 is a relay node only: it receives LoRa packets from Node 1, appends its own sensor reading, and forwards to Node 3\. It requires no WiFi or Bluetooth, making the Arduino Nano (\~₱90) a better fit than a second ESP32 (\~₱220). Both share the Arduino IDE, keeping the team on a single development environment.

### Ra-02 SX1278 LoRa 433MHz

At ₱200 each, the Ra-02 is locally available and cost-effective. The 433MHz band offers better obstacle penetration than higher bands, with a practical hop range of 1–3 km in semi-open provincial terrain. Connects via SPI to both the ESP32 and Arduino Nano.

### JSN-SR04T Waterproof Ultrasonic Sensor

IP67-rated. Mounts above the waterline pointing downward and measures distance to the water surface (range: 20–600 cm). Operated in UART mode for clean, jitter-free readings. Distance-based output is essential for computing rate of rise, the primary indicator in flash flood early warning.

### A7680C 4G LTE Module — Modular, Node 3

Designed as a future upgrade via a UART header on Node 3\. The firmware uses a `#ifdef USE_4G` compile flag — when the module is present, it handles both HTTP posting and AT command SMS; when absent, ESP32 WiFi handles both. This keeps the prototype cost under ₱3,000 while documenting a clear cellular upgrade path.

## VI. Sensor Strategy

The JSN-SR04T is the primary sensor at all three nodes, reading water level every 30 seconds. Two optional add-ons are planned for post-prototype iterations:

* **YF-S201 flow sensor** at Nodes 1 and 2 — detects canal blockages. A sudden drop in flow rate alongside rising water level indicates clogging rather than rainfall, enabling more targeted maintenance alerts to the relevant barangay.  
* **Tipping bucket rain gauge** at Node 1 — provides upstream rainfall context, allowing pre-emptive alerts before downstream water levels react.

## VII. Edge Impulse ML Integration

Standard threshold-based alerting does not distinguish between a gradual rainfall event and a sudden surge, nor does it adapt to seasonal baseline shifts. Edge Impulse addresses this by training a time-series anomaly detection model on actual sensor data collected during prototype deployment.

### Model design

The model takes a rolling window of water level readings from all three nodes as input and classifies each window as either normal flow or anomalous surge. Training data will be collected during the prototype's initial deployment period, with anomalous events labeled manually as they occur.

### Deployment

Edge Impulse's EON Compiler generates C++ inference code that runs directly on the ESP32 in Node 3 (240MHz, 520KB SRAM — sufficient for a lightweight time-series classifier). Inference runs locally on the device with no cloud call required, making the alert system resilient to network congestion during actual flood events.

### Alert integration

Node 3 runs inference every 60 seconds. An ML surge flag is cross-checked against the rule-based threshold logic before an alert is issued, reducing false positives. During the initial data collection phase, the system operates on rule-based thresholds only; the ML layer activates once sufficient labeled data has been collected and a validated model deployed.

### Academic significance

Integrating Edge Impulse moves the project from a sensor data-logging system to an intelligent edge IoT system — aligning with current research directions in edge AI and making the capstone academically distinguishable from simpler threshold-based flood monitors.

## VIII. Flash Flood Alert Logic

| Level | Trigger condition | Response |
| :---- | :---- | :---- |
| Yellow | Rise \> 5 cm in 10 minutes | SMS to barangay tanod and DRRMO officer |
| Orange | Rise \> 15 cm in 10 minutes | SMS to barangay captain and pre-registered residents |
| Red | Rise \> 30 cm in 10 min, or Edge Impulse surge flag | SMS all contacts \+ dashboard alarm |

Thresholds are configurable constants in the Node 3 firmware, adjustable per deployment site and canal dimensions.

## 

## IX. Software Stack

| Layer | Tool | Cost |
| :---- | :---- | ----: |
| Firmware | Arduino IDE \+ LoRaMesher / RadioHead library | Free |
| ML model | Edge Impulse (free academic tier) | Free |
| Backend | Firebase Realtime Database | Free tier |
| Dashboard | Three.js (3D canal sim) \+ Chart.js (time-series) | Free |
| Hosting | Firebase Hosting or Vercel | Free tier |
| SMS alerts | A7680C native AT+CMGS — no API, no per-message fee | Free |
| Data SIM (Node 3\) | Prepaid load, \~₱50/month for 30s interval payloads | \~₱50/month |

## X. Bill of Materials and Costing

| Component | N1 | N2 | N3 | Unit cost | Total |
| ----- | :---: | :---: | :---: | ----: | ----: |
| Wemos Lolin32 v1 | ✓ | — | ✓ | ₱220 | ₱440 |
| Arduino Nano | — | ✓ | — | ₱90 | ₱90 |
| Ra-02 SX1278 LoRa | ✓ | ✓ | ✓ | ₱200 | ₱600 |
| JSN-SR04T sensor | ✓ | ✓ | ✓ | ₱170 | ₱510 |
| 18650 LiPo cell | ✓ | ✓ | ✓ | ₱100 | ₱300 |
| TP4056 module | — | ✓ | — | ₱25 | ₱25 |
| 5V 1W solar panel | ✓ | ✓ | ✓ | ₱120 | ₱360 |
| IP65 ABS enclosure | ✓ | ✓ | ✓ | ₱80 | ₱240 |
| Misc (Wiring/PCBs) | ✓ | ✓ | ✓ | ₱75 | ₱225 |
| **Core prototype total** |  |  |  |  | **₱2,790** |

Note: Lolin32 boards include onboard LiPo charging circuits; TP4056 is only required on Node 2 (Arduino Nano).

## XI. Power Design

Lolin32 nodes connect directly to a LiPo cell via JST connector — the onboard charger handles solar-to-battery management. At 30-second reading intervals with deep sleep between samples, average current draw is approximately 15mA, giving over 8 days of standalone battery life. With partial solar exposure (e.g., overcast), the system is indefinitely self-sustaining. For shaded canal locations, spare charged 18650 cells maintained by barangay tanod staff serve as a practical backup.

## XII. Risk Assessment

| Risk | Mitigation |
| :---- | :---- |
| LoRa signal blockage | Site survey before installation; 433MHz performs better than higher bands around obstacles |
| Solar shading | Spare charged batteries maintained by barangay; deep sleep minimizes power draw |
| Node submersion | Mount 1.5m+ above canal floor; IP65 provides short-term water resistance |
| WiFi dependency | Modular A7680C slot allows immediate cellular upgrade |
| ML false positives | Cross-check with rule-based thresholds before issuing SMS alerts |
| Insufficient ML data | Deploy threshold-only alerting first; collect 2–4 weeks of data before enabling ML inference |

## XIII. Scope and Limitations

This prototype validates the complete end-to-end pipeline across 3 nodes: sensor reading, LoRa hop relay, cloud ingestion, dashboard visualization, SMS alerting, and edge ML inference. It does not cover an entire waterway system — that would require a formal site survey and additional nodes. Each additional relay node requires only an Arduino Nano \+ Ra-02 \+ JSN-SR04T at approximately ₱460.

## XIV. Future Enhancements

* Cellular independence via A7680C installation on Node 3  
* Expanded node coverage upstream using the same relay hardware  
* Canal clog detection via YF-S201 flow sensor pairs  
* Retraining pipeline for Edge Impulse model as labeled flood data accumulates  
* Integration with PAGASA forecast API for predictive pre-alerting  
* Mobile companion app using the same Firebase Realtime Database backend

