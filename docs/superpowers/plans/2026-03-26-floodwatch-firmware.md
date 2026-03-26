# FloodWatch Firmware Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build complete field node firmware — sensors, LoRa mesh, actuators, deep sleep, auto-detection — as a single Arduino codebase that runs on all Lolin32 nodes.

**Architecture:** Layer-by-layer build. Test sketches validate each sensor/component in isolation (Tasks 1–7). Then the production firmware is composed from modules: config, detect, sensors, lora_comm, actuators, power (Tasks 8–14). Final integration test with two physical nodes (Task 15).

**Tech Stack:** Arduino IDE, ESP32 board package, LoRaMesher library, RadioLib (SX1278 driver)

**Adapted for embedded:** Traditional TDD doesn't apply to microcontroller firmware. "Testing" here means: upload sketch → verify via Serial Monitor output and physical observation. Each test sketch is a standalone `.ino` that proves one component works before it's composed into the production firmware.

---

## File Structure

```
firmware/
├── test_sketches/                    # Layer 1-2: standalone test sketches
│   ├── test_jsn_sr04t/
│   │   └── test_jsn_sr04t.ino        # Water level sensor validation
│   ├── test_yf_s201/
│   │   └── test_yf_s201.ino          # Flow sensor validation
│   ├── test_ky003_rain/
│   │   └── test_ky003_rain.ino       # Rain gauge validation
│   ├── test_buzzer_leds/
│   │   └── test_buzzer_leds.ino      # Actuator pattern validation
│   ├── test_battery_adc/
│   │   └── test_battery_adc.ino      # Battery voltage reading
│   ├── test_lora_tx/
│   │   └── test_lora_tx.ino          # LoRa point-to-point transmit
│   └── test_lora_rx/
│       └── test_lora_rx.ino          # LoRa point-to-point receive
│
└── FloodWatch_Node/                  # Layer 3-6: production firmware
    ├── FloodWatch_Node.ino           # Main setup/loop, sleep/wake cycle
    ├── config.h                      # Pin assignments, constants, packet format
    ├── detect.h                      # Auto-detection function declarations
    ├── detect.cpp                    # Auto-detect plugged sensors, read DIP switch
    ├── sensors.h                     # Sensor reading declarations
    ├── sensors.cpp                   # Read JSN-SR04T, YF-S201, KY-003, battery
    ├── actuators.h                   # Actuator control declarations
    ├── actuators.cpp                 # Buzzer patterns, LED states
    ├── lora_comm.h                   # LoRa communication declarations
    ├── lora_comm.cpp                 # LoRaMesher init, uplink, downlink
    ├── power.h                       # Power management declarations
    └── power.cpp                     # Transistor switching, deep sleep
```

---

## Layer 1 — Individual Sensor Tinkering

> **Parallel-friendly:** Tasks 1-4 can be split across two groups.
> Group 1: Tasks 1 + 2 (water level + flow). Group 2: Tasks 3 + 4 (rain + actuators).
> Task 5 (battery) is quick and can be done by either group.

### Task 1: JSN-SR04T Water Level Sensor Test

**Files:**
- Create: `firmware/test_sketches/test_jsn_sr04t/test_jsn_sr04t.ino`

**What you need:** Lolin32, JSN-SR04T sensor, 4 dupont wires

**Wiring:**
- JSN-SR04T VCC → Lolin32 5V
- JSN-SR04T GND → Lolin32 GND
- JSN-SR04T TX → Lolin32 GPIO 16 (Serial2 RX)
- JSN-SR04T RX → Lolin32 GPIO 17 (Serial2 TX)

- [ ] **Step 1: Create the test sketch**

```cpp
// test_jsn_sr04t.ino
// Validates JSN-SR04T ultrasonic sensor in UART mode (Mode 4)
// Expected: prints distance in cm to Serial Monitor every 500ms

#define JSN_RX_PIN 16  // ESP32 receives on this pin (sensor TX)
#define JSN_TX_PIN 17  // ESP32 transmits on this pin (sensor RX)

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, JSN_RX_PIN, JSN_TX_PIN);

  Serial.println("=== JSN-SR04T UART Mode Test ===");
  Serial.println("Point sensor at a surface 20-600cm away.");
  Serial.println("Readings every 500ms...");
  Serial.println();
}

uint16_t readDistanceCm() {
  // JSN-SR04T UART mode: send 0x55 to trigger, wait for 4-byte response
  // Response format: [0xFF] [high_byte] [low_byte] [checksum]
  // Distance in mm = (high_byte << 8) | low_byte

  // Flush any stale data
  while (Serial2.available()) Serial2.read();

  // Send trigger byte
  Serial2.write(0x55);

  // Wait for response (up to 100ms)
  unsigned long start = millis();
  while (Serial2.available() < 4) {
    if (millis() - start > 100) {
      Serial.println("ERROR: No response from sensor (timeout)");
      return 0;
    }
  }

  uint8_t header = Serial2.read();
  if (header != 0xFF) {
    Serial.print("ERROR: Bad header byte: 0x");
    Serial.println(header, HEX);
    // Flush remaining bytes
    while (Serial2.available()) Serial2.read();
    return 0;
  }

  uint8_t highByte = Serial2.read();
  uint8_t lowByte = Serial2.read();
  uint8_t checksum = Serial2.read();

  // Verify checksum: (0xFF + high + low) & 0xFF should equal checksum
  uint8_t calcChecksum = (0xFF + highByte + lowByte) & 0xFF;
  if (calcChecksum != checksum) {
    Serial.println("ERROR: Checksum mismatch");
    return 0;
  }

  uint16_t distanceMm = (highByte << 8) | lowByte;
  return distanceMm / 10;  // Convert mm to cm
}

void loop() {
  uint16_t cm = readDistanceCm();
  if (cm > 0) {
    Serial.print("Distance: ");
    Serial.print(cm);
    Serial.print(" cm");
    if (cm < 20) Serial.print("  [WARNING: below min range]");
    if (cm > 600) Serial.print("  [WARNING: above max range]");
    Serial.println();
  }
  delay(500);
}
```

- [ ] **Step 2: Upload and verify**

1. Open in Arduino IDE, select board "WEMOS LOLIN32", correct COM port
2. Upload sketch
3. Open Serial Monitor at 115200 baud
4. Point sensor at a flat surface (wall, floor, table)
5. Expected output: `Distance: XX cm` readings that change as you move the sensor
6. Verify: readings are stable (not jumping wildly), match a ruler measurement within ~1cm

- [ ] **Step 3: Physical mounting test**

1. Hold sensor at the angle you'd mount it (pointing straight down)
2. Move a flat surface (book, board) underneath at different heights
3. Note: does reading become unstable at shallow angles? Find the mounting sweet spot
4. Test with water surface if possible (bucket, basin)

- [ ] **Step 4: Commit**

```bash
git add firmware/test_sketches/test_jsn_sr04t/
git commit -m "test: add JSN-SR04T ultrasonic sensor UART test sketch"
```

---

### Task 2: YF-S201 Flow Sensor Test

**Files:**
- Create: `firmware/test_sketches/test_yf_s201/test_yf_s201.ino`

**What you need:** Lolin32, YF-S201 flow sensor, 3 dupont wires, water source + tubing

**Wiring:**
- YF-S201 Red → Lolin32 5V
- YF-S201 Black → Lolin32 GND
- YF-S201 Yellow (signal) → Lolin32 GPIO 27

- [ ] **Step 1: Create the test sketch**

```cpp
// test_yf_s201.ino
// Validates YF-S201 hall-effect flow sensor via interrupt counting
// Expected: prints flow rate in L/min and total liters every second

#define FLOW_PIN 27

volatile uint32_t pulseCount = 0;

void IRAM_ATTR flowPulseISR() {
  pulseCount++;
}

void setup() {
  Serial.begin(115200);
  pinMode(FLOW_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(FLOW_PIN), flowPulseISR, RISING);

  Serial.println("=== YF-S201 Flow Sensor Test ===");
  Serial.println("Run water through the sensor.");
  Serial.println("Readings every second...");
  Serial.println();
}

float totalLiters = 0.0;
unsigned long lastTime = 0;

void loop() {
  if (millis() - lastTime >= 1000) {
    // Disable interrupts briefly to read and reset count
    noInterrupts();
    uint32_t count = pulseCount;
    pulseCount = 0;
    interrupts();

    // YF-S201 calibration: ~7.5 pulses per second per L/min
    // Flow rate (L/min) = pulses per second / 7.5
    float flowRate = count / 7.5;
    totalLiters += flowRate / 60.0;  // Add this second's volume

    Serial.print("Pulses: ");
    Serial.print(count);
    Serial.print("  Flow: ");
    Serial.print(flowRate, 2);
    Serial.print(" L/min");
    Serial.print("  Total: ");
    Serial.print(totalLiters, 3);
    Serial.println(" L");

    lastTime = millis();
  }
}
```

- [ ] **Step 2: Upload and verify**

1. Upload sketch, open Serial Monitor at 115200
2. With no water flowing: should show `Pulses: 0  Flow: 0.00 L/min`
3. Blow through the sensor (or run water): pulse count should increase
4. Verify readings respond to flow changes

- [ ] **Step 3: Calibration check**

1. Run a known volume of water (e.g., 1 liter measured with a bottle) through the sensor
2. Compare the `Total` reading to the actual volume
3. If off by more than 10%, note the correction factor for later

- [ ] **Step 4: Commit**

```bash
git add firmware/test_sketches/test_yf_s201/
git commit -m "test: add YF-S201 flow sensor interrupt test sketch"
```

---

### Task 3: KY-003 Rain Gauge Test

**Files:**
- Create: `firmware/test_sketches/test_ky003_rain/test_ky003_rain.ino`

**What you need:** Lolin32, KY-003 hall sensor, small magnet, 3 dupont wires

**Wiring:**
- KY-003 VCC → Lolin32 3.3V
- KY-003 GND → Lolin32 GND
- KY-003 Signal → Lolin32 GPIO 25

- [ ] **Step 1: Create the test sketch**

```cpp
// test_ky003_rain.ino
// Validates KY-003 hall sensor as tipping bucket rain gauge trigger
// Each magnet pass = one bucket tip = one rainfall increment
// Expected: counts tips and prints accumulated rainfall

#define RAIN_PIN 25

// Calibration: each tip = this many mm of rain
// Depends on your DIY bucket size. Start with 0.2mm and adjust.
#define MM_PER_TIP 0.2

volatile uint32_t tipCount = 0;
volatile unsigned long lastTipTime = 0;

void IRAM_ATTR rainTipISR() {
  // Debounce: ignore tips within 200ms of each other
  unsigned long now = millis();
  if (now - lastTipTime > 200) {
    tipCount++;
    lastTipTime = now;
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(RAIN_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(RAIN_PIN), rainTipISR, FALLING);

  Serial.println("=== KY-003 Rain Gauge Test ===");
  Serial.println("Pass a magnet near the sensor to simulate bucket tips.");
  Serial.println("Each tip = one rainfall increment.");
  Serial.println();
}

uint32_t lastDisplayedCount = 0;

void loop() {
  noInterrupts();
  uint32_t count = tipCount;
  interrupts();

  if (count != lastDisplayedCount) {
    float rainfallMm = count * MM_PER_TIP;
    Serial.print("Tips: ");
    Serial.print(count);
    Serial.print("  Rainfall: ");
    Serial.print(rainfallMm, 1);
    Serial.println(" mm");
    lastDisplayedCount = count;
  }

  delay(50);
}
```

- [ ] **Step 2: Upload and verify**

1. Upload sketch, open Serial Monitor at 115200
2. Wave a magnet near the KY-003 sensor
3. Each pass should increment the tip count by 1
4. Verify debounce works: fast magnet swipes don't double-count

- [ ] **Step 3: Tipping bucket mechanism test**

If you've built the DIY tipping bucket:
1. Mount KY-003 next to the pivot point, magnet on the bucket arm
2. Pour measured water into the funnel
3. Count tips vs expected for your bucket volume
4. Adjust `MM_PER_TIP` if needed

- [ ] **Step 4: Commit**

```bash
git add firmware/test_sketches/test_ky003_rain/
git commit -m "test: add KY-003 hall sensor rain gauge test sketch"
```

---

### Task 4: Buzzer + LED Actuator Test

**Files:**
- Create: `firmware/test_sketches/test_buzzer_leds/test_buzzer_leds.ino`

**What you need:** Lolin32, active buzzer module, 2N2222A transistor, 1kΩ resistor, 3x 5mm LEDs (red, yellow, blue), 3x 220Ω resistors, breadboard

**Wiring:**
- Buzzer VCC → Lolin32 5V
- Buzzer GND → 2N2222A collector. 2N2222A emitter → GND. 2N2222A base → 1kΩ → GPIO 32
- LED Red anode → 220Ω → GPIO 33. Cathode → GND
- LED Yellow anode → 220Ω → GPIO 12. Cathode → GND
- LED Blue anode → 220Ω → GPIO 13. Cathode → GND

- [ ] **Step 1: Create the test sketch**

```cpp
// test_buzzer_leds.ino
// Cycles through all alert patterns: normal, yellow, orange, red, clog
// Each pattern runs for 5 seconds before advancing

#define BUZZER_PIN 32
#define LED_RED    33
#define LED_YELLOW 12
#define LED_BLUE   13

// Alert states (matches downlink protocol)
enum AlertState : uint8_t {
  ALERT_NORMAL = 0,
  ALERT_YELLOW = 1,
  ALERT_ORANGE = 2,
  ALERT_RED    = 3,
  ALERT_CLOG   = 4
};

const char* alertNames[] = {"NORMAL", "YELLOW", "ORANGE", "RED", "CLOG"};

void setLEDs(bool red, bool yellow, bool blue) {
  digitalWrite(LED_RED, red ? HIGH : LOW);
  digitalWrite(LED_YELLOW, yellow ? HIGH : LOW);
  digitalWrite(LED_BLUE, blue ? HIGH : LOW);
}

void beep(uint16_t onMs, uint16_t offMs) {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(onMs);
  digitalWrite(BUZZER_PIN, LOW);
  delay(offMs);
}

void runPattern(AlertState state, unsigned long durationMs) {
  unsigned long start = millis();

  // Set LEDs for this state
  switch (state) {
    case ALERT_NORMAL: setLEDs(false, false, false); break;
    case ALERT_YELLOW: setLEDs(false, true, false); break;
    case ALERT_ORANGE: setLEDs(true, true, false); break;  // Red + Yellow = Orange
    case ALERT_RED:    setLEDs(true, false, false); break;
    case ALERT_CLOG:   setLEDs(false, false, true); break;
  }

  while (millis() - start < durationMs) {
    switch (state) {
      case ALERT_NORMAL:
        delay(100);  // Silent
        break;

      case ALERT_YELLOW:
        // Short single beep, repeating
        beep(100, 900);
        break;

      case ALERT_ORANGE:
        // Fast double beep, repeating
        beep(80, 80);
        beep(80, 760);
        break;

      case ALERT_RED:
        // Continuous tone
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100);
        break;

      case ALERT_CLOG:
        // Slow triple beep, repeating
        beep(150, 150);
        beep(150, 150);
        beep(150, 1250);
        break;
    }
  }

  // Clean up
  digitalWrite(BUZZER_PIN, LOW);
  setLEDs(false, false, false);
}

void setup() {
  Serial.begin(115200);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  pinMode(LED_YELLOW, OUTPUT);
  pinMode(LED_BLUE, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);
  setLEDs(false, false, false);

  Serial.println("=== Buzzer + LED Pattern Test ===");
  Serial.println("Cycles through all 5 alert states, 5 seconds each.");
  Serial.println();
}

void loop() {
  for (int i = 0; i <= 4; i++) {
    AlertState state = (AlertState)i;
    Serial.print(">>> Pattern: ");
    Serial.print(alertNames[i]);
    Serial.println(" (5 seconds)");

    runPattern(state, 5000);

    Serial.println("    Done. 2 second pause...");
    delay(2000);
  }
  Serial.println("=== Cycle complete. Restarting... ===\n");
}
```

- [ ] **Step 2: Upload and verify**

1. Upload sketch, open Serial Monitor at 115200
2. Watch and listen as it cycles through each pattern:
   - NORMAL: silent, all LEDs off
   - YELLOW: short beep repeating, yellow LED on
   - ORANGE: fast double beep, red + yellow LEDs on
   - RED: continuous tone, red LED on
   - CLOG: slow triple beep, blue LED on
3. Verify each pattern is distinguishable by sound alone (field crew won't be looking at LEDs)

- [ ] **Step 3: Commit**

```bash
git add firmware/test_sketches/test_buzzer_leds/
git commit -m "test: add buzzer and LED alert pattern test sketch"
```

---

### Task 5: Battery ADC Test

**Files:**
- Create: `firmware/test_sketches/test_battery_adc/test_battery_adc.ino`

**What you need:** Lolin32, 2x resistors for voltage divider (100kΩ + 100kΩ, or similar), 18650 battery in holder

**Wiring:**
- Battery positive → 100kΩ → GPIO 35 → 100kΩ → GND
- (This halves the voltage so a 4.2V battery reads ~2.1V, safe for ESP32 ADC)

- [ ] **Step 1: Create the test sketch**

```cpp
// test_battery_adc.ino
// Reads battery voltage via voltage divider on GPIO 35
// Expected: prints millivolts, percentage estimate, and low-battery flag

#define BATTERY_PIN 35

// Voltage divider ratio: R1 = R2 = 100kΩ → ratio = 2.0
// Adjust if your resistors differ
#define DIVIDER_RATIO 2.0

// ESP32 ADC calibration (12-bit, 0-4095)
// Default attenuation: 0-1.1V range. We'll set 11dB for 0-3.3V range.
#define ADC_MAX 4095
#define ADC_VREF 3.3

// LiPo voltage thresholds
#define BATT_FULL_MV   4200
#define BATT_NOMINAL_MV 3700
#define BATT_LOW_MV    3300
#define BATT_CRITICAL_MV 3000

uint16_t readBatteryMv() {
  // Average 16 readings for stability
  uint32_t sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += analogRead(BATTERY_PIN);
    delay(2);
  }
  float adcAvg = sum / 16.0;

  // Convert ADC to voltage, then multiply by divider ratio
  float voltage = (adcAvg / ADC_MAX) * ADC_VREF * DIVIDER_RATIO;
  return (uint16_t)(voltage * 1000);  // Return millivolts
}

uint8_t batteryPercent(uint16_t mv) {
  if (mv >= BATT_FULL_MV) return 100;
  if (mv <= BATT_CRITICAL_MV) return 0;
  return (uint8_t)((mv - BATT_CRITICAL_MV) * 100 / (BATT_FULL_MV - BATT_CRITICAL_MV));
}

void setup() {
  Serial.begin(115200);
  analogSetAttenuation(ADC_11db);  // 0-3.3V range
  pinMode(BATTERY_PIN, INPUT);

  Serial.println("=== Battery ADC Test ===");
  Serial.println("Reads battery voltage every second via GPIO 35 divider.");
  Serial.println();
}

void loop() {
  uint16_t mv = readBatteryMv();
  uint8_t pct = batteryPercent(mv);
  bool lowBattery = mv < BATT_LOW_MV;

  Serial.print("Battery: ");
  Serial.print(mv);
  Serial.print(" mV  (");
  Serial.print(pct);
  Serial.print("%)");
  if (lowBattery) Serial.print("  [LOW BATTERY]");
  if (mv < BATT_CRITICAL_MV) Serial.print("  [CRITICAL]");
  Serial.println();

  delay(1000);
}
```

- [ ] **Step 2: Upload and verify**

1. Upload sketch, open Serial Monitor at 115200
2. With USB power only (no battery): reading will be 0 or erratic — that's expected
3. Connect a charged 18650: should read 3700–4200 mV
4. Compare reading against a multimeter on the battery terminals
5. If readings are off by more than 100mV, adjust `DIVIDER_RATIO`

- [ ] **Step 3: Commit**

```bash
git add firmware/test_sketches/test_battery_adc/
git commit -m "test: add battery voltage ADC test sketch"
```

---

## Layer 2 — Point-to-Point LoRa

> Both sketches run simultaneously on two separate Lolin32 boards.
> You need 2 Lolin32s + 2 Ra-02 modules + 2 antennas for this layer.

### Task 6: LoRa Point-to-Point TX

**Files:**
- Create: `firmware/test_sketches/test_lora_tx/test_lora_tx.ino`

**What you need:** Lolin32 #1, Ra-02 module, 12dBi antenna + SMA-uFL adapter, dupont wires

**Wiring (same for TX and RX):**
- Ra-02 VCC → Lolin32 3.3V
- Ra-02 GND → Lolin32 GND
- Ra-02 SCK → GPIO 18
- Ra-02 MISO → GPIO 19
- Ra-02 MOSI → GPIO 23
- Ra-02 NSS → GPIO 5
- Ra-02 RST → GPIO 14
- Ra-02 DIO0 → GPIO 26

**CRITICAL:** Always attach the antenna before powering the Ra-02. Transmitting without an antenna can damage the module.

- [ ] **Step 1: Install RadioLib library**

In Arduino IDE: Sketch → Include Library → Manage Libraries → search "RadioLib" → Install

- [ ] **Step 2: Create the TX test sketch**

```cpp
// test_lora_tx.ino
// Sends a test packet every 3 seconds on 433MHz
// Pair with test_lora_rx.ino on a second board

#include <RadioLib.h>

// SX1278 pin mapping for Ra-02 on Lolin32
#define LORA_NSS   5
#define LORA_DIO0  26
#define LORA_RST   14
#define LORA_DIO1  -1  // Not connected

SX1278 radio = new Module(LORA_NSS, LORA_DIO0, LORA_RST, LORA_DIO1);

uint32_t packetNum = 0;

void setup() {
  Serial.begin(115200);
  Serial.println("=== LoRa TX Test ===");

  // Initialize SX1278
  // Frequency: 433.0 MHz, Bandwidth: 125 kHz, SF: 9, CR: 4/7
  Serial.print("Initializing LoRa... ");
  int state = radio.begin(433.0, 125.0, 9, 7);
  if (state == RADIOLIB_ERR_NONE) {
    Serial.println("OK");
  } else {
    Serial.print("FAILED, code ");
    Serial.println(state);
    Serial.println("Check wiring! Halting.");
    while (true) delay(1000);
  }

  // Set output power (2-17 dBm for SX1278)
  radio.setOutputPower(17);

  Serial.println("Transmitting every 3 seconds...");
  Serial.println();
}

void loop() {
  // Build a simple test message
  char msg[64];
  snprintf(msg, sizeof(msg), "FloodWatch TX test #%lu", packetNum);

  Serial.print("Sending: ");
  Serial.print(msg);

  int state = radio.transmit(msg);
  if (state == RADIOLIB_ERR_NONE) {
    Serial.print("  [OK, ");
    Serial.print(radio.getDataRate());
    Serial.println(" bps]");
  } else {
    Serial.print("  [FAILED, code ");
    Serial.print(state);
    Serial.println("]");
  }

  packetNum++;
  delay(3000);
}
```

- [ ] **Step 3: Upload to board #1**

1. Connect Lolin32 #1 (with Ra-02 + antenna attached)
2. Upload sketch
3. Open Serial Monitor — should print "Initializing LoRa... OK" then "Sending..." every 3s
4. If init fails: double-check SPI wiring, especially NSS (GPIO 5) and RST (GPIO 14)

- [ ] **Step 4: Commit**

```bash
git add firmware/test_sketches/test_lora_tx/
git commit -m "test: add LoRa point-to-point TX test sketch"
```

---

### Task 7: LoRa Point-to-Point RX

**Files:**
- Create: `firmware/test_sketches/test_lora_rx/test_lora_rx.ino`

**What you need:** Lolin32 #2, Ra-02 module, 12dBi antenna, same wiring as Task 6

- [ ] **Step 1: Create the RX test sketch**

```cpp
// test_lora_rx.ino
// Listens for packets from test_lora_tx.ino and prints them
// Run on a second Lolin32 board simultaneously

#include <RadioLib.h>

#define LORA_NSS   5
#define LORA_DIO0  26
#define LORA_RST   14
#define LORA_DIO1  -1

SX1278 radio = new Module(LORA_NSS, LORA_DIO0, LORA_RST, LORA_DIO1);

void setup() {
  Serial.begin(115200);
  Serial.println("=== LoRa RX Test ===");

  // Must match TX settings exactly
  Serial.print("Initializing LoRa... ");
  int state = radio.begin(433.0, 125.0, 9, 7);
  if (state == RADIOLIB_ERR_NONE) {
    Serial.println("OK");
  } else {
    Serial.print("FAILED, code ");
    Serial.println(state);
    while (true) delay(1000);
  }

  Serial.println("Listening for packets...");
  Serial.println();
}

void loop() {
  String received;
  int state = radio.receive(received);

  if (state == RADIOLIB_ERR_NONE) {
    Serial.print("Received: ");
    Serial.print(received);
    Serial.print("  [RSSI: ");
    Serial.print(radio.getRSSI());
    Serial.print(" dBm, SNR: ");
    Serial.print(radio.getSNR());
    Serial.println(" dB]");
  } else if (state != RADIOLIB_ERR_RX_TIMEOUT) {
    Serial.print("Receive error, code ");
    Serial.println(state);
  }
}
```

- [ ] **Step 2: Upload to board #2 and verify both boards**

1. Upload to Lolin32 #2 (with Ra-02 + antenna)
2. Open Serial Monitor on both boards (use two Arduino IDE windows or two COM ports)
3. Board #1 (TX) should print "Sending..." every 3s
4. Board #2 (RX) should print "Received: FloodWatch TX test #N" with RSSI/SNR values
5. RSSI closer to 0 = stronger signal. Typical indoor: -30 to -80 dBm

- [ ] **Step 3: Range test**

1. Move boards apart — different rooms, different floors
2. Note RSSI at each distance
3. Verify packets still arrive through walls (433MHz should penetrate well)
4. Find the point where packets start dropping

- [ ] **Step 4: Commit**

```bash
git add firmware/test_sketches/test_lora_rx/
git commit -m "test: add LoRa point-to-point RX test sketch"
```

---

## Layer 3–6 — Production Firmware

### Task 8: config.h — Pin Definitions and Constants

**Files:**
- Create: `firmware/FloodWatch_Node/config.h`

- [ ] **Step 1: Create config.h**

```cpp
// config.h
// Pin assignments, packet format, and constants for all FloodWatch nodes.
// This file is the single source of truth for hardware configuration.

#ifndef FLOODWATCH_CONFIG_H
#define FLOODWATCH_CONFIG_H

#include <Arduino.h>

// ── LoRa SPI (Ra-02 SX1278) ──────────────────────────────
#define PIN_LORA_SCK    18
#define PIN_LORA_MISO   19
#define PIN_LORA_MOSI   23
#define PIN_LORA_NSS     5
#define PIN_LORA_RST    14
#define PIN_LORA_DIO0   26

// ── LoRa Power Switching (2N2222A) ───────────────────────
#define PIN_LORA_PWR     4   // HIGH = Ra-02 powered on

// ── Sensors ──────────────────────────────────────────────
#define PIN_JSN_RX      16   // Serial2 RX (sensor TX)
#define PIN_JSN_TX      17   // Serial2 TX (sensor RX)
#define PIN_FLOW        27   // YF-S201 signal (interrupt)
#define PIN_RAIN        25   // KY-003 signal (interrupt)

// ── Actuators ────────────────────────────────────────────
#define PIN_BUZZER      32   // Via 2N2222A transistor
#define PIN_LED_RED     33
#define PIN_LED_YELLOW  12
#define PIN_LED_BLUE    13

// ── Battery ADC ──────────────────────────────────────────
#define PIN_BATTERY     35   // Voltage divider input
#define BATT_DIVIDER_RATIO 2.0
#define BATT_LOW_MV     3300
#define BATT_CRITICAL_MV 3000
#define BATT_FULL_MV    4200
#define BATT_CHARGE_THRESHOLD_MV 4400  // Above this = solar is charging

// ── DIP Switch (Node ID) ────────────────────────────────
#define PIN_DIP_BIT0    21   // LSB
#define PIN_DIP_BIT1    22
#define PIN_DIP_BIT2    34   // Input-only, needs external pull-up
#define PIN_DIP_BIT3    36   // Input-only, needs external pull-up

// ── LoRa Radio Config ───────────────────────────────────
#define LORA_FREQUENCY  433.0  // MHz
#define LORA_BANDWIDTH  125.0  // kHz
#define LORA_SF         9      // Spreading factor
#define LORA_CR         7      // Coding rate 4/7

// ── Timing ───────────────────────────────────────────────
#define SLEEP_DURATION_US  30000000ULL  // 30 seconds in microseconds
#define LORA_BOOT_DELAY_MS 10
#define LORA_RX_WINDOW_MS  1000
#define SENSOR_READ_TIMEOUT_MS 100

// ── Packet Format ────────────────────────────────────────

// Uplink: node → base station (10 bytes)
struct __attribute__((packed)) UplinkPacket {
  uint8_t  node_id;          // From DIP switch
  uint8_t  packet_type;      // 0x01 = sensor, 0x02 = heartbeat
  uint16_t water_level_cm;   // JSN-SR04T reading
  uint16_t flow_rate_dLmin;  // YF-S201 in deci-liters/min (0 if absent)
  uint8_t  rain_tips;        // Tips since last packet (0 if absent)
  uint16_t battery_mv;       // Battery voltage
  uint8_t  flags;            // Bit 0: has_flow, Bit 1: has_rain, Bit 2: low_battery
};

#define FLAG_HAS_FLOW    0x01
#define FLAG_HAS_RAIN    0x02
#define FLAG_LOW_BATTERY 0x04

#define PKT_TYPE_SENSOR    0x01
#define PKT_TYPE_HEARTBEAT 0x02

// Downlink: base station → node (2 bytes)
struct __attribute__((packed)) DownlinkPacket {
  uint8_t target_node_id;  // 0xFF = broadcast
  uint8_t alert_state;     // 0=normal, 1=yellow, 2=orange, 3=red, 4=clog
};

// Alert state enum
enum AlertState : uint8_t {
  ALERT_NORMAL = 0,
  ALERT_YELLOW = 1,
  ALERT_ORANGE = 2,
  ALERT_RED    = 3,
  ALERT_CLOG   = 4
};

// ── Node Configuration (populated at boot by detect.cpp) ─
struct NodeConfig {
  uint8_t node_id;
  bool has_flow;
  bool has_rain;
  bool has_lora;
  bool has_solar;
};

#endif // FLOODWATCH_CONFIG_H
```

- [ ] **Step 2: Commit**

```bash
git add firmware/FloodWatch_Node/config.h
git commit -m "feat: add config.h with pin definitions and packet format"
```

---

### Task 9: detect module — Auto-Detection and DIP Switch

**Files:**
- Create: `firmware/FloodWatch_Node/detect.h`
- Create: `firmware/FloodWatch_Node/detect.cpp`

- [ ] **Step 1: Create detect.h**

```cpp
// detect.h
#ifndef FLOODWATCH_DETECT_H
#define FLOODWATCH_DETECT_H

#include "config.h"

// Reads DIP switch and probes sensors to populate NodeConfig.
// Call once in setup() before initializing any other module.
NodeConfig detectHardware();

// Print detected config to Serial for debugging
void printNodeConfig(const NodeConfig& cfg);

#endif
```

- [ ] **Step 2: Create detect.cpp**

```cpp
// detect.cpp
#include "detect.h"

static uint8_t readDipSwitch() {
  // GPIO 21, 22 have internal pull-ups enabled
  // GPIO 34, 36 need external 10kΩ pull-ups on the PCB
  pinMode(PIN_DIP_BIT0, INPUT_PULLUP);
  pinMode(PIN_DIP_BIT1, INPUT_PULLUP);
  pinMode(PIN_DIP_BIT2, INPUT);  // No internal pull-up on 34
  pinMode(PIN_DIP_BIT3, INPUT);  // No internal pull-up on 36

  delay(5);  // Let pins settle

  // DIP switch ON = pulled to GND = reads LOW = bit is 1
  uint8_t id = 0;
  if (digitalRead(PIN_DIP_BIT0) == LOW) id |= 0x01;
  if (digitalRead(PIN_DIP_BIT1) == LOW) id |= 0x02;
  if (digitalRead(PIN_DIP_BIT2) == LOW) id |= 0x04;
  if (digitalRead(PIN_DIP_BIT3) == LOW) id |= 0x08;

  return id;
}

static bool detectJsnSr04t() {
  // Try UART ping: send 0x55, wait for 4-byte response
  Serial2.begin(9600, SERIAL_8N1, PIN_JSN_RX, PIN_JSN_TX);
  delay(50);

  // Flush
  while (Serial2.available()) Serial2.read();

  Serial2.write(0x55);

  unsigned long start = millis();
  while (Serial2.available() < 4) {
    if (millis() - start > SENSOR_READ_TIMEOUT_MS) {
      Serial2.end();
      return false;
    }
  }

  uint8_t header = Serial2.read();
  // Don't end Serial2 — leave it open if sensor is present
  if (header == 0xFF) {
    // Flush remaining bytes
    while (Serial2.available()) Serial2.read();
    return true;
  }

  Serial2.end();
  return false;
}

static bool detectFlowSensor() {
  // YF-S201: when present, the hall sensor output is either LOW or pulsing.
  // When absent with pull-up, pin reads steady HIGH.
  // Read 10 samples over 50ms — if ALL are HIGH, sensor is absent.
  pinMode(PIN_FLOW, INPUT_PULLUP);
  delay(10);

  int highCount = 0;
  for (int i = 0; i < 10; i++) {
    if (digitalRead(PIN_FLOW) == HIGH) highCount++;
    delay(5);
  }

  return highCount < 10;  // At least one LOW = sensor is present
}

static bool detectRainSensor() {
  // Same logic as flow sensor detection
  pinMode(PIN_RAIN, INPUT_PULLUP);
  delay(10);

  int highCount = 0;
  for (int i = 0; i < 10; i++) {
    if (digitalRead(PIN_RAIN) == HIGH) highCount++;
    delay(5);
  }

  return highCount < 10;
}

static bool detectSolar(uint16_t batteryMv) {
  // If battery voltage is above charge threshold, solar panel is connected
  // and actively charging. This is a rough heuristic.
  return batteryMv > BATT_CHARGE_THRESHOLD_MV;
}

NodeConfig detectHardware() {
  NodeConfig cfg;
  cfg.node_id  = readDipSwitch();
  cfg.has_lora  = true;   // Verified later during LoRa init
  cfg.has_flow  = detectFlowSensor();
  cfg.has_rain  = detectRainSensor();

  // Battery voltage needed for solar detection
  // Quick ADC read (detailed reading is in sensors.cpp)
  analogSetAttenuation(ADC_11db);
  pinMode(PIN_BATTERY, INPUT);
  uint32_t sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += analogRead(PIN_BATTERY);
    delay(2);
  }
  float voltage = (sum / 16.0 / 4095.0) * 3.3 * BATT_DIVIDER_RATIO;
  uint16_t mv = (uint16_t)(voltage * 1000);

  cfg.has_solar = detectSolar(mv);

  return cfg;
}

void printNodeConfig(const NodeConfig& cfg) {
  Serial.println("=== Node Configuration ===");
  Serial.print("  Node ID:    ");
  Serial.println(cfg.node_id);
  Serial.print("  LoRa:       ");
  Serial.println(cfg.has_lora ? "detected" : "NOT FOUND");
  Serial.print("  Flow (S201):");
  Serial.println(cfg.has_flow ? "detected" : "not connected");
  Serial.print("  Rain (K003):");
  Serial.println(cfg.has_rain ? "detected" : "not connected");
  Serial.print("  Solar:      ");
  Serial.println(cfg.has_solar ? "charging" : "not detected");
  Serial.println("==========================");
}
```

- [ ] **Step 3: Create a minimal .ino to test detection**

Create a temporary `firmware/FloodWatch_Node/FloodWatch_Node.ino` with just detection:

```cpp
// FloodWatch_Node.ino (temporary — detection test only)
#include "config.h"
#include "detect.h"

NodeConfig nodeCfg;

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("\n\nFloodWatch Node — Detection Test");

  nodeCfg = detectHardware();
  printNodeConfig(nodeCfg);
}

void loop() {
  delay(5000);
  Serial.println("Re-scanning...");
  nodeCfg = detectHardware();
  printNodeConfig(nodeCfg);
}
```

- [ ] **Step 4: Upload and verify detection**

1. Upload with NO sensors plugged in → all should show "not connected"
2. Plug in JSN-SR04T → re-scan should detect it
3. Plug in YF-S201 → should detect flow sensor
4. Change DIP switch positions → node ID should change
5. Verify detection is reliable (run 5+ scans, no false positives)

- [ ] **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/config.h firmware/FloodWatch_Node/detect.h firmware/FloodWatch_Node/detect.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add hardware auto-detection and DIP switch reading"
```

---

### Task 10: sensors module

**Files:**
- Create: `firmware/FloodWatch_Node/sensors.h`
- Create: `firmware/FloodWatch_Node/sensors.cpp`

- [ ] **Step 1: Create sensors.h**

```cpp
// sensors.h
#ifndef FLOODWATCH_SENSORS_H
#define FLOODWATCH_SENSORS_H

#include "config.h"

// Initialize sensors based on detected config.
// Call once in setup() after detectHardware().
void sensorsInit(const NodeConfig& cfg);

// Read all sensors and build an uplink packet.
// Skips absent sensors (fields set to 0, flags cleared).
UplinkPacket sensorsRead(const NodeConfig& cfg);

// Reset per-cycle counters (rain tips since last read)
void sensorsResetCycle();

#endif
```

- [ ] **Step 2: Create sensors.cpp**

```cpp
// sensors.cpp
#include "sensors.h"

// ── Flow sensor interrupt ────────────────────────────────
static volatile uint32_t _flowPulseCount = 0;

static void IRAM_ATTR flowPulseISR() {
  _flowPulseCount++;
}

// ── Rain gauge interrupt ─────────────────────────────────
static volatile uint32_t _rainTipCount = 0;
static volatile unsigned long _lastTipTime = 0;

static void IRAM_ATTR rainTipISR() {
  unsigned long now = millis();
  if (now - _lastTipTime > 200) {  // 200ms debounce
    _rainTipCount++;
    _lastTipTime = now;
  }
}

// ── Tracking between reads ───────────────────────────────
static uint32_t _lastFlowPulses = 0;
static uint32_t _lastRainTips = 0;

void sensorsInit(const NodeConfig& cfg) {
  // JSN-SR04T is always expected (core sensor)
  Serial2.begin(9600, SERIAL_8N1, PIN_JSN_RX, PIN_JSN_TX);

  if (cfg.has_flow) {
    pinMode(PIN_FLOW, INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(PIN_FLOW), flowPulseISR, RISING);
  }

  if (cfg.has_rain) {
    pinMode(PIN_RAIN, INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(PIN_RAIN), rainTipISR, FALLING);
  }

  // Battery ADC
  analogSetAttenuation(ADC_11db);
  pinMode(PIN_BATTERY, INPUT);
}

static uint16_t readWaterLevelCm() {
  // Flush stale data
  while (Serial2.available()) Serial2.read();

  Serial2.write(0x55);

  unsigned long start = millis();
  while (Serial2.available() < 4) {
    if (millis() - start > SENSOR_READ_TIMEOUT_MS) return 0;
  }

  uint8_t header = Serial2.read();
  if (header != 0xFF) {
    while (Serial2.available()) Serial2.read();
    return 0;
  }

  uint8_t highByte = Serial2.read();
  uint8_t lowByte  = Serial2.read();
  uint8_t checksum = Serial2.read();

  uint8_t calcChecksum = (0xFF + highByte + lowByte) & 0xFF;
  if (calcChecksum != checksum) return 0;

  uint16_t distanceMm = (highByte << 8) | lowByte;
  return distanceMm / 10;
}

static uint16_t readFlowRateDlMin() {
  // Get pulses since last read
  noInterrupts();
  uint32_t pulses = _flowPulseCount;
  _flowPulseCount = 0;
  interrupts();

  uint32_t deltaPulses = pulses;

  // YF-S201: 7.5 pulses/sec per L/min
  // Over 30 seconds: total pulses / 30 = pulses/sec
  // pulses/sec / 7.5 = L/min
  // Multiply by 10 for deci-liters/min (dL/min) to keep integer precision
  // dL/min = (deltaPulses / 30.0 / 7.5) * 10.0
  float lPerMin = deltaPulses / 30.0 / 7.5;
  return (uint16_t)(lPerMin * 10);  // dL/min
}

static uint8_t readRainTips() {
  noInterrupts();
  uint32_t tips = _rainTipCount;
  _rainTipCount = 0;
  interrupts();

  return (tips > 255) ? 255 : (uint8_t)tips;
}

static uint16_t readBatteryMv() {
  uint32_t sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += analogRead(PIN_BATTERY);
    delay(2);
  }
  float voltage = (sum / 16.0 / 4095.0) * 3.3 * BATT_DIVIDER_RATIO;
  return (uint16_t)(voltage * 1000);
}

UplinkPacket sensorsRead(const NodeConfig& cfg) {
  UplinkPacket pkt;
  pkt.node_id        = cfg.node_id;
  pkt.packet_type    = PKT_TYPE_SENSOR;
  pkt.water_level_cm = readWaterLevelCm();
  pkt.flow_rate_dLmin = cfg.has_flow ? readFlowRateDlMin() : 0;
  pkt.rain_tips      = cfg.has_rain ? readRainTips() : 0;
  pkt.battery_mv     = readBatteryMv();

  pkt.flags = 0;
  if (cfg.has_flow)                   pkt.flags |= FLAG_HAS_FLOW;
  if (cfg.has_rain)                   pkt.flags |= FLAG_HAS_RAIN;
  if (pkt.battery_mv < BATT_LOW_MV)  pkt.flags |= FLAG_LOW_BATTERY;

  return pkt;
}

void sensorsResetCycle() {
  noInterrupts();
  _flowPulseCount = 0;
  _rainTipCount = 0;
  interrupts();
}
```

- [ ] **Step 3: Update .ino to test sensor readings**

Replace the temporary `FloodWatch_Node.ino`:

```cpp
// FloodWatch_Node.ino (temporary — sensor test)
#include "config.h"
#include "detect.h"
#include "sensors.h"

NodeConfig nodeCfg;

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("\n\nFloodWatch Node — Sensor Test");

  nodeCfg = detectHardware();
  printNodeConfig(nodeCfg);

  sensorsInit(nodeCfg);
  Serial.println("Sensors initialized. Reading every 5 seconds...\n");
}

void loop() {
  UplinkPacket pkt = sensorsRead(nodeCfg);

  Serial.println("--- Sensor Reading ---");
  Serial.print("  Node ID:     "); Serial.println(pkt.node_id);
  Serial.print("  Water level: "); Serial.print(pkt.water_level_cm); Serial.println(" cm");
  Serial.print("  Flow rate:   "); Serial.print(pkt.flow_rate_dLmin / 10.0, 1); Serial.println(" L/min");
  Serial.print("  Rain tips:   "); Serial.println(pkt.rain_tips);
  Serial.print("  Battery:     "); Serial.print(pkt.battery_mv); Serial.println(" mV");
  Serial.print("  Flags:       0x"); Serial.println(pkt.flags, HEX);
  Serial.println();

  delay(5000);
}
```

- [ ] **Step 4: Upload and verify**

1. Upload with JSN-SR04T connected → water level should show valid cm reading
2. If flow/rain sensors connected → those fields should populate
3. Absent sensors should show 0 with flags unset
4. Battery reading should be reasonable

- [ ] **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/sensors.h firmware/FloodWatch_Node/sensors.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add sensors module with water level, flow, rain, battery"
```

---

### Task 11: actuators module

**Files:**
- Create: `firmware/FloodWatch_Node/actuators.h`
- Create: `firmware/FloodWatch_Node/actuators.cpp`

- [ ] **Step 1: Create actuators.h**

```cpp
// actuators.h
#ifndef FLOODWATCH_ACTUATORS_H
#define FLOODWATCH_ACTUATORS_H

#include "config.h"

// Initialize buzzer and LED pins. Call once in setup().
void actuatorsInit();

// Set the current alert state. Updates LEDs immediately.
// Buzzer pattern runs in actuatorsUpdate().
void actuatorsSetState(AlertState state);

// Get the current alert state.
AlertState actuatorsGetState();

// Call this repeatedly in loop() to drive buzzer patterns.
// Non-blocking — uses millis() timing internally.
void actuatorsUpdate();

// Turn off everything (before deep sleep).
void actuatorsOff();

#endif
```

- [ ] **Step 2: Create actuators.cpp**

```cpp
// actuators.cpp
#include "actuators.h"

static AlertState _currentState = ALERT_NORMAL;
static unsigned long _patternStart = 0;

void actuatorsInit() {
  pinMode(PIN_BUZZER, OUTPUT);
  pinMode(PIN_LED_RED, OUTPUT);
  pinMode(PIN_LED_YELLOW, OUTPUT);
  pinMode(PIN_LED_BLUE, OUTPUT);
  actuatorsOff();
}

static void setLEDs(bool red, bool yellow, bool blue) {
  digitalWrite(PIN_LED_RED, red ? HIGH : LOW);
  digitalWrite(PIN_LED_YELLOW, yellow ? HIGH : LOW);
  digitalWrite(PIN_LED_BLUE, blue ? HIGH : LOW);
}

void actuatorsSetState(AlertState state) {
  _currentState = state;
  _patternStart = millis();

  switch (state) {
    case ALERT_NORMAL: setLEDs(false, false, false); break;
    case ALERT_YELLOW: setLEDs(false, true, false);  break;
    case ALERT_ORANGE: setLEDs(true, true, false);   break;
    case ALERT_RED:    setLEDs(true, false, false);   break;
    case ALERT_CLOG:   setLEDs(false, false, true);   break;
  }

  if (state == ALERT_NORMAL) {
    digitalWrite(PIN_BUZZER, LOW);
  }
}

AlertState actuatorsGetState() {
  return _currentState;
}

void actuatorsUpdate() {
  if (_currentState == ALERT_NORMAL) return;

  unsigned long elapsed = millis() - _patternStart;

  switch (_currentState) {
    case ALERT_YELLOW: {
      // Short single beep: 100ms on, 900ms off (1s cycle)
      unsigned long phase = elapsed % 1000;
      digitalWrite(PIN_BUZZER, phase < 100 ? HIGH : LOW);
      break;
    }

    case ALERT_ORANGE: {
      // Fast double beep: 80on, 80off, 80on, 760off (1s cycle)
      unsigned long phase = elapsed % 1000;
      if (phase < 80)        digitalWrite(PIN_BUZZER, HIGH);
      else if (phase < 160)  digitalWrite(PIN_BUZZER, LOW);
      else if (phase < 240)  digitalWrite(PIN_BUZZER, HIGH);
      else                   digitalWrite(PIN_BUZZER, LOW);
      break;
    }

    case ALERT_RED: {
      // Continuous tone
      digitalWrite(PIN_BUZZER, HIGH);
      break;
    }

    case ALERT_CLOG: {
      // Slow triple beep: 150on,150off,150on,150off,150on,1250off (2s cycle)
      unsigned long phase = elapsed % 2000;
      if (phase < 150)       digitalWrite(PIN_BUZZER, HIGH);
      else if (phase < 300)  digitalWrite(PIN_BUZZER, LOW);
      else if (phase < 450)  digitalWrite(PIN_BUZZER, HIGH);
      else if (phase < 600)  digitalWrite(PIN_BUZZER, LOW);
      else if (phase < 750)  digitalWrite(PIN_BUZZER, HIGH);
      else                   digitalWrite(PIN_BUZZER, LOW);
      break;
    }

    default:
      break;
  }
}

void actuatorsOff() {
  _currentState = ALERT_NORMAL;
  digitalWrite(PIN_BUZZER, LOW);
  setLEDs(false, false, false);
}
```

- [ ] **Step 3: Test by adding to .ino**

Add actuator test to `FloodWatch_Node.ino` — cycle through states via Serial commands:

```cpp
// FloodWatch_Node.ino (temporary — sensor + actuator test)
#include "config.h"
#include "detect.h"
#include "sensors.h"
#include "actuators.h"

NodeConfig nodeCfg;

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("\n\nFloodWatch Node — Sensor + Actuator Test");

  nodeCfg = detectHardware();
  printNodeConfig(nodeCfg);

  sensorsInit(nodeCfg);
  actuatorsInit();

  Serial.println("Send 0-4 via Serial to set alert state:");
  Serial.println("  0=normal, 1=yellow, 2=orange, 3=red, 4=clog");
  Serial.println();
}

unsigned long lastSensorRead = 0;

void loop() {
  // Check for Serial commands
  if (Serial.available()) {
    char c = Serial.read();
    if (c >= '0' && c <= '4') {
      AlertState state = (AlertState)(c - '0');
      actuatorsSetState(state);
      Serial.print("Alert state set to: ");
      Serial.println(c - '0');
    }
  }

  // Drive buzzer patterns (non-blocking)
  actuatorsUpdate();

  // Read sensors every 5 seconds
  if (millis() - lastSensorRead > 5000) {
    UplinkPacket pkt = sensorsRead(nodeCfg);
    Serial.print("[sensors] level=");
    Serial.print(pkt.water_level_cm);
    Serial.print("cm  batt=");
    Serial.print(pkt.battery_mv);
    Serial.println("mV");
    lastSensorRead = millis();
  }
}
```

- [ ] **Step 4: Upload and verify**

1. Upload, open Serial Monitor
2. Type `0` through `4` to test each alert pattern
3. Verify buzzer patterns match spec, LEDs light correct colors
4. Verify `actuatorsUpdate()` is non-blocking (sensor readings still print)

- [ ] **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/actuators.h firmware/FloodWatch_Node/actuators.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add actuators module with non-blocking buzzer patterns"
```

---

### Task 12: lora_comm module — LoRaMesher Integration

**Files:**
- Create: `firmware/FloodWatch_Node/lora_comm.h`
- Create: `firmware/FloodWatch_Node/lora_comm.cpp`

**Prerequisites:** Install LoRaMesher library in Arduino IDE. Check https://github.com/LoRaMesher/LoRaMesher for installation instructions and current API. The code below follows the documented API pattern — verify against the version you install.

- [ ] **Step 1: Create lora_comm.h**

```cpp
// lora_comm.h
#ifndef FLOODWATCH_LORA_COMM_H
#define FLOODWATCH_LORA_COMM_H

#include "config.h"

// Initialize LoRaMesher on the Ra-02 module.
// Returns true if SX1278 was detected and mesh started.
bool loraInit();

// Send an uplink packet into the mesh.
// LoRaMesher handles routing to the gateway node.
bool loraSendUplink(const UplinkPacket& pkt);

// Check for and return any pending downlink command.
// Returns true if a command was received, populates `out`.
bool loraReceiveDownlink(DownlinkPacket& out);

// Call periodically to let LoRaMesher process its internal queue.
void loraUpdate();

#endif
```

- [ ] **Step 2: Create lora_comm.cpp**

```cpp
// lora_comm.cpp
//
// IMPORTANT: This code follows the LoRaMesher library's documented API.
// When you install LoRaMesher, check its examples/ folder and verify
// that the function signatures match. The library is under active
// development and the API may change between versions.

#include "lora_comm.h"
#include <LoraMesher.h>

static LoraMesher& mesh = LoraMesher::getInstance();
static DownlinkPacket _pendingDownlink;
static volatile bool _hasDownlink = false;

// Callback: LoRaMesher calls this when a packet arrives for us
static void onReceive(uint8_t* payload, uint8_t length) {
  if (length == sizeof(DownlinkPacket)) {
    memcpy(&_pendingDownlink, payload, sizeof(DownlinkPacket));
    _hasDownlink = true;
  }
}

bool loraInit() {
  // Configure the radio parameters before starting
  LoraMesher::LoraMesherConfig config;
  config.loraCs  = PIN_LORA_NSS;
  config.loraRst = PIN_LORA_RST;
  config.loraIrq = PIN_LORA_DIO0;
  config.loraIo1 = -1;  // DIO1 not connected
  config.freq    = LORA_FREQUENCY;
  config.bw      = LORA_BANDWIDTH;
  config.sf      = LORA_SF;
  config.cr      = LORA_CR;

  // Initialize the mesh
  mesh.begin(config);

  // Set receive callback
  mesh.setReceiveAppDataTaskHandle(onReceive);

  // Start the mesh networking tasks
  mesh.start();

  Serial.print("LoRaMesher started. Local address: 0x");
  Serial.println(mesh.getLocalAddress(), HEX);

  return true;
}

bool loraSendUplink(const UplinkPacket& pkt) {
  // Send to the gateway (address 0x0000 = broadcast / gateway)
  // LoRaMesher will route through intermediate nodes if needed
  uint8_t* data = (uint8_t*)&pkt;
  uint8_t len = sizeof(UplinkPacket);

  // Use reliable send which retries on failure
  mesh.createPacketAndSend(
    BROADCAST_ADDR,  // LoRaMesher broadcast — gateway picks it up
    data,
    len
  );

  return true;
}

bool loraReceiveDownlink(DownlinkPacket& out) {
  if (!_hasDownlink) return false;

  memcpy(&out, &_pendingDownlink, sizeof(DownlinkPacket));
  _hasDownlink = false;
  return true;
}

void loraUpdate() {
  // LoRaMesher runs its own FreeRTOS tasks, so this is a no-op
  // for most versions. Included for future compatibility.
}
```

> **NOTE FOR THE TEAM:** The LoRaMesher API above is based on the library's documented examples. When you install the library, open its example sketches (`File → Examples → LoRaMesher`) and compare the init sequence. If the API differs from what's written here, update `lora_comm.cpp` to match. The key things to verify: `LoraMesherConfig` struct fields, `begin()` vs `init()` method names, and `createPacketAndSend()` parameters.

- [ ] **Step 3: Test with two boards**

Update `FloodWatch_Node.ino` to include LoRa:

```cpp
// FloodWatch_Node.ino (temporary — full integration test minus power)
#include "config.h"
#include "detect.h"
#include "sensors.h"
#include "actuators.h"
#include "lora_comm.h"

NodeConfig nodeCfg;

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("\n\nFloodWatch Node — LoRa Mesh Test");

  // Power on LoRa module
  pinMode(PIN_LORA_PWR, OUTPUT);
  digitalWrite(PIN_LORA_PWR, HIGH);
  delay(LORA_BOOT_DELAY_MS);

  nodeCfg = detectHardware();
  printNodeConfig(nodeCfg);

  sensorsInit(nodeCfg);
  actuatorsInit();

  if (loraInit()) {
    nodeCfg.has_lora = true;
    Serial.println("LoRa mesh active.");
  } else {
    nodeCfg.has_lora = false;
    Serial.println("LoRa init FAILED.");
  }

  Serial.println("Sending sensor data every 30 seconds...");
  Serial.println("Send 0-4 via Serial to simulate downlink alert.");
  Serial.println();
}

unsigned long lastSend = 0;

void loop() {
  // Simulate downlink via Serial (for testing without a real base station)
  if (Serial.available()) {
    char c = Serial.read();
    if (c >= '0' && c <= '4') {
      actuatorsSetState((AlertState)(c - '0'));
      Serial.print("Simulated alert: "); Serial.println(c - '0');
    }
  }

  // Check for real LoRa downlink
  DownlinkPacket dl;
  if (loraReceiveDownlink(dl)) {
    if (dl.target_node_id == nodeCfg.node_id || dl.target_node_id == 0xFF) {
      actuatorsSetState((AlertState)dl.alert_state);
      Serial.print("LoRa downlink received! Alert state: ");
      Serial.println(dl.alert_state);
    }
  }

  // Drive buzzer patterns
  actuatorsUpdate();

  // Send uplink every 30 seconds
  if (millis() - lastSend > 30000) {
    UplinkPacket pkt = sensorsRead(nodeCfg);
    Serial.print("[TX] level=");
    Serial.print(pkt.water_level_cm);
    Serial.print("cm  batt=");
    Serial.print(pkt.battery_mv);
    Serial.println("mV");

    if (nodeCfg.has_lora) {
      loraSendUplink(pkt);
      Serial.println("[TX] Packet sent to mesh.");
    }

    lastSend = millis();
  }

  loraUpdate();
}
```

- [ ] **Step 4: Upload to both boards and verify mesh**

1. Upload to both Lolin32 boards (both with Ra-02 + antenna)
2. Both should show "LoRa mesh active" with different local addresses
3. Each board should send packets every 30 seconds
4. Open Serial Monitor on both — verify packets are being sent
5. At this point, without a base station, you won't see received data, but LoRaMesher should log routing table updates

- [ ] **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/lora_comm.h firmware/FloodWatch_Node/lora_comm.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add LoRaMesher communication module"
```

---

### Task 13: power module — Deep Sleep and Transistor Switching

**Files:**
- Create: `firmware/FloodWatch_Node/power.h`
- Create: `firmware/FloodWatch_Node/power.cpp`

- [ ] **Step 1: Create power.h**

```cpp
// power.h
#ifndef FLOODWATCH_POWER_H
#define FLOODWATCH_POWER_H

#include "config.h"

// Initialize power control pins (transistor gates).
void powerInit();

// Power on the LoRa module via 2N2222A.
void powerLoraOn();

// Power off the LoRa module via 2N2222A.
void powerLoraOff();

// Enter deep sleep for SLEEP_DURATION_US microseconds.
// All GPIO states are lost on wake — setup() runs again.
void powerDeepSleep();

// Check if this boot was a wake from deep sleep (vs. power-on/reset).
bool powerIsWakeFromSleep();

#endif
```

- [ ] **Step 2: Create power.cpp**

```cpp
// power.cpp
#include "power.h"
#include <esp_sleep.h>

void powerInit() {
  pinMode(PIN_LORA_PWR, OUTPUT);
  digitalWrite(PIN_LORA_PWR, LOW);  // Start with LoRa off

  pinMode(PIN_BUZZER, OUTPUT);
  digitalWrite(PIN_BUZZER, LOW);
}

void powerLoraOn() {
  digitalWrite(PIN_LORA_PWR, HIGH);
  delay(LORA_BOOT_DELAY_MS);
}

void powerLoraOff() {
  digitalWrite(PIN_LORA_PWR, LOW);
}

void powerDeepSleep() {
  // Turn off everything before sleeping
  powerLoraOff();
  digitalWrite(PIN_BUZZER, LOW);
  digitalWrite(PIN_LED_RED, LOW);
  digitalWrite(PIN_LED_YELLOW, LOW);
  digitalWrite(PIN_LED_BLUE, LOW);

  // Configure wake timer
  esp_sleep_enable_timer_wakeup(SLEEP_DURATION_US);

  Serial.println("[power] Entering deep sleep...");
  Serial.flush();

  esp_deep_sleep_start();
  // Execution stops here. On wake, setup() runs from the top.
}

bool powerIsWakeFromSleep() {
  return esp_sleep_get_wakeup_cause() == ESP_SLEEP_WAKEUP_TIMER;
}
```

- [ ] **Step 3: Commit**

```bash
git add firmware/FloodWatch_Node/power.h firmware/FloodWatch_Node/power.cpp
git commit -m "feat: add power module with deep sleep and transistor switching"
```

---

### Task 14: Main Firmware — FloodWatch_Node.ino (Final)

**Files:**
- Modify: `firmware/FloodWatch_Node/FloodWatch_Node.ino`

- [ ] **Step 1: Write the final main sketch**

```cpp
// FloodWatch_Node.ino
// Main firmware for all FloodWatch field sensor nodes.
// Behavior is determined at boot by auto-detecting plugged sensors.
//
// Sleep/wake cycle:
//   wake → power LoRa → read sensors → send uplink → check downlink
//   → if alert: stay awake running pattern
//   → if normal: deep sleep 30s → repeat

#include "config.h"
#include "detect.h"
#include "sensors.h"
#include "actuators.h"
#include "lora_comm.h"
#include "power.h"

NodeConfig nodeCfg;
AlertState currentAlert = ALERT_NORMAL;

void setup() {
  Serial.begin(115200);

  bool woke = powerIsWakeFromSleep();
  if (woke) {
    Serial.println("\n[wake] Timer wakeup");
  } else {
    Serial.println("\n\n=== FloodWatch Node ===");
  }

  // Initialize power control first
  powerInit();

  // Detect hardware
  nodeCfg = detectHardware();
  if (!woke) {
    printNodeConfig(nodeCfg);
  }

  // Initialize modules
  sensorsInit(nodeCfg);
  actuatorsInit();

  // Power on LoRa and start mesh
  powerLoraOn();
  if (loraInit()) {
    nodeCfg.has_lora = true;
  } else {
    nodeCfg.has_lora = false;
    Serial.println("[ERROR] LoRa init failed!");
  }

  // ── Main cycle: read → send → receive → decide ────────

  // 1. Read sensors
  UplinkPacket pkt = sensorsRead(nodeCfg);
  Serial.print("[data] level=");
  Serial.print(pkt.water_level_cm);
  Serial.print("cm flow=");
  Serial.print(pkt.flow_rate_dLmin);
  Serial.print(" rain=");
  Serial.print(pkt.rain_tips);
  Serial.print(" batt=");
  Serial.print(pkt.battery_mv);
  Serial.println("mV");

  // 2. Send uplink
  if (nodeCfg.has_lora) {
    loraSendUplink(pkt);
    Serial.println("[lora] Uplink sent");
  }

  // 3. Listen for downlink commands
  unsigned long rxStart = millis();
  bool gotCommand = false;
  while (millis() - rxStart < LORA_RX_WINDOW_MS) {
    loraUpdate();
    DownlinkPacket dl;
    if (loraReceiveDownlink(dl)) {
      if (dl.target_node_id == nodeCfg.node_id || dl.target_node_id == 0xFF) {
        currentAlert = (AlertState)dl.alert_state;
        actuatorsSetState(currentAlert);
        Serial.print("[lora] Alert command: ");
        Serial.println(dl.alert_state);
        gotCommand = true;
        break;
      }
    }
    delay(10);
  }

  // 4. Decide: sleep or stay awake
  if (currentAlert == ALERT_NORMAL) {
    // No alert — power down and sleep
    powerLoraOff();
    actuatorsOff();
    powerDeepSleep();
    // Does not return. setup() runs again on wake.
  }

  // Alert active — stay awake (falls through to loop)
  Serial.println("[alert] Staying awake for alert pattern.");
}

void loop() {
  // Only reaches here during active alert state.
  // Drive buzzer pattern (non-blocking).
  actuatorsUpdate();

  // Periodically check for updated commands (every 5 seconds)
  static unsigned long lastCheck = 0;
  if (millis() - lastCheck > 5000) {
    loraUpdate();
    DownlinkPacket dl;
    if (loraReceiveDownlink(dl)) {
      if (dl.target_node_id == nodeCfg.node_id || dl.target_node_id == 0xFF) {
        currentAlert = (AlertState)dl.alert_state;
        actuatorsSetState(currentAlert);
        Serial.print("[lora] Updated alert: ");
        Serial.println(dl.alert_state);

        // If alert cleared, go to sleep on next cycle
        if (currentAlert == ALERT_NORMAL) {
          Serial.println("[alert] Cleared. Sleeping...");
          powerDeepSleep();
        }
      }
    }
    lastCheck = millis();
  }

  // Also read and send sensor data while awake (every 30s)
  static unsigned long lastSend = 0;
  if (millis() - lastSend > 30000) {
    UplinkPacket pkt = sensorsRead(nodeCfg);
    if (nodeCfg.has_lora) {
      loraSendUplink(pkt);
    }
    lastSend = millis();
  }
}
```

- [ ] **Step 2: Upload and verify full cycle**

1. Upload to a Lolin32 with Ra-02 + JSN-SR04T connected
2. Open Serial Monitor — should see detection, sensor reading, uplink sent
3. If no alert received → should print "Entering deep sleep..." and go quiet
4. After 30 seconds → should wake and repeat
5. Verify power consumption: during sleep, current should drop to microamps (if you have a multimeter in series with battery)

- [ ] **Step 3: Test alert override**

Without a base station sending real downlinks, test the alert-stays-awake behavior:
1. Temporarily modify the code to simulate receiving an alert (hardcode `currentAlert = ALERT_YELLOW` after the RX window)
2. Verify the node stays awake, runs the buzzer pattern, and continues reading sensors
3. Remove the hardcoded alert, re-upload

- [ ] **Step 4: Commit**

```bash
git add firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: complete main firmware with sleep/wake cycle and alert override"
```

---

### Task 15: Two-Node Integration Test

**Files:** No new files — this is a verification task.

**What you need:** Both Lolin32 boards fully wired (Ra-02 + antenna + JSN-SR04T minimum), DIP switches set to different IDs.

- [ ] **Step 1: Set up Node A (DIP switch = 1)**

1. Set DIP switch to `0001` (node ID 1)
2. Connect JSN-SR04T + flow sensor + rain gauge (if available)
3. Upload `FloodWatch_Node` firmware
4. Open Serial Monitor — verify detection shows all connected sensors

- [ ] **Step 2: Set up Node B (DIP switch = 2)**

1. Set DIP switch to `0010` (node ID 2)
2. Connect JSN-SR04T only
3. Upload same `FloodWatch_Node` firmware
4. Open Serial Monitor — verify detection shows JSN-SR04T only, flow/rain absent

- [ ] **Step 3: Verify LoRa mesh communication**

1. Power both nodes simultaneously
2. Both should initialize LoRaMesher and show mesh addresses
3. Wait for routing table updates in Serial output (LoRaMesher logs these)
4. Both nodes should be sending uplink packets every 30 seconds
5. Verify both nodes appear in each other's routing tables

- [ ] **Step 4: Verify deep sleep cycle**

1. Watch both nodes cycle: wake → read → send → sleep → wake
2. Verify the 30-second interval is consistent
3. If you have a multimeter: measure current during sleep (should be ~10µA)

- [ ] **Step 5: Document results**

Create a simple test log noting:
- Node IDs detected correctly? (yes/no)
- Sensors auto-detected correctly? (yes/no)
- LoRa packets sent/received? (yes/no, RSSI values)
- Deep sleep working? (yes/no, measured current if available)
- Any issues encountered and how they were resolved

- [ ] **Step 6: Commit test log**

```bash
git add docs/test-log-firmware.md
git commit -m "docs: add firmware integration test results"
```

---

## What's Next

After this firmware is working on two nodes:

1. **Base Station** — Orange Pi software to receive LoRa packets (via USB serial from a gateway node), run alert logic, send SMS, push to Firebase
2. **Dashboard** — Three.js canal visualization + Chart.js time-series on Firebase data
3. **ML Pipeline** — Edge Impulse model training after 2–4 weeks of sensor data collection

Each is a separate spec → plan → implementation cycle.
