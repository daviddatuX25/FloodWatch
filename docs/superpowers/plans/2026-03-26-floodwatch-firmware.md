# FloodWatch Firmware Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build complete field node firmware — sensors, LoRa mesh, actuators, deep sleep, auto-detection — as a single Arduino codebase that runs on all Lolin32 nodes.

**Architecture:** Layer-by-layer build. Test sketches validate each sensor/component in isolation (Tasks 1–7). Then the production firmware is composed from modules: config, detect, sensors, lora_comm, actuators, power (Tasks 8–14). Final integration test with two physical nodes (Task 15).

**Tech Stack:** Arduino IDE, ESP32 board package, LoRaMesher library, RadioLib (SX1278 driver), Wokwi simulator (pre-hardware validation)

**Adapted for embedded:** Traditional TDD doesn't apply to microcontroller firmware. Testing is two-stage: first simulate in Wokwi to validate logic, Serial output, and GPIO behavior without hardware risk; then upload to physical board for final verification with real sensors. Components Wokwi cannot simulate (JSN-SR04T UART, HB100 microwave Doppler radar, KY-003, SX1278 LoRa, LoRaMesher) are stubbed or skipped in simulation and validated hardware-only.

---

## File Structure

```
firmware/
├── test_sketches/                    # Layer 1-2: standalone test sketches
│   ├── test_jsn_sr04t/
│   │   └── test_jsn_sr04t.ino        # Water level sensor validation
│   ├── test_hb100_velocity/
│   │   └── test_hb100_velocity.ino   # HB100 Doppler radar surface velocity validation
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
    ├── sensors.cpp                   # Read JSN-SR04T, HB100 (ADC+FFT), KY-003, battery
    ├── actuators.h                   # Actuator control declarations
    ├── actuators.cpp                 # Buzzer patterns, LED states
    ├── lora_comm.h                   # LoRa communication declarations
    ├── lora_comm.cpp                 # LoRaMesher init, uplink, downlink
    ├── power.h                       # Power management declarations
    └── power.cpp                     # Transistor switching, deep sleep
```

```
wokwi/
├── task04_buzzer_leds/
│   ├── diagram.json                  # ESP32 + 3 LEDs + buzzer
│   └── wokwi.toml
├── task05_battery_adc/
│   ├── diagram.json                  # ESP32 + potentiometer on GPIO 35
│   └── wokwi.toml
└── task14_main_firmware/
    ├── diagram.json                  # ESP32 + LEDs + buzzer + pot + pushbuttons
    └── wokwi.toml
```

---

## Layer 1 — Individual Sensor Tinkering

> **Parallel-friendly:** Tasks 1-4 can be split across two groups.
> Group 1: Tasks 1 + 2 (water level + HB100 velocity). Group 2: Tasks 3 + 4 (rain gauge + actuators).
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
- **Step 1: Create the test sketch**

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

- **Step 2: Simulate in Wokwi (limited)**

Wokwi does not support JSN-SR04T. To validate Serial output and loop logic:

1. Comment out the `Serial2.write(0x55)` trigger and response-read block in `readDistanceCm()`
2. Hardcode a return value: `return 150;`
3. Run in Wokwi — verify Serial prints "Distance: 150 cm" every 500ms
4. Restore original code before uploading to hardware

UART protocol correctness requires the physical sensor.

- **Step 3: Upload and verify**

1. Open in Arduino IDE, select board "WEMOS LOLIN32", correct COM port
2. Upload sketch
3. Open Serial Monitor at 115200 baud
4. Point sensor at a flat surface (wall, floor, table)
5. Expected output: `Distance: XX cm` readings that change as you move the sensor
6. Verify: readings are stable (not jumping wildly), match a ruler measurement within ~1cm

- **Step 3: Physical mounting test**

1. Hold sensor at the angle you'd mount it (pointing straight down)
2. Move a flat surface (book, board) underneath at different heights
3. Note: does reading become unstable at shallow angles? Find the mounting sweet spot
4. Test with water surface if possible (bucket, basin)

- **Step 4: Commit**

```bash
git add firmware/test_sketches/test_jsn_sr04t/
git commit -m "test: add JSN-SR04T ultrasonic sensor UART test sketch"
```

---

### Task 2: HB100 Doppler Radar Surface Velocity Test

**Files:**

- Create: `firmware/test_sketches/test_hb100_velocity/test_hb100_velocity.ino`

> **Library required:** arduinoFFT by Enrique Condes — install via Library Manager (search "arduinoFFT").

**What you need:** Lolin32, HB100 module, LM358 op-amp, resistors (100kΩ ×2, 10kΩ ×2), capacitors (10µF, 0.1µF), breadboard, 5V power, dupont wires

**Wiring — HB100:**

- HB100 VCC → Lolin32 5V
- HB100 GND → Lolin32 GND
- HB100 IF pin → LM358 input (pin 3)

**Wiring — LM358 amplifier (non-inverting, gain ~660, low-pass filter):**

- LM358 pin 8 (V+) → 5V
- LM358 pin 4 (GND) → GND
- LM358 pin 3 (+IN) → HB100 IF pin (with 0.1µF cap to GND to filter RF)
- LM358 pin 2 (−IN) → junction of 100kΩ (to output) and 10kΩ (to GND)
- LM358 pin 1 (OUT) → Lolin32 GPIO 27 (ADC)
- 10µF bypass cap across 5V supply pins

- **Step 1: Create the test sketch**

```cpp
// test_hb100_velocity.ino
// Validates HB100 X-Band Doppler radar via ADC sampling + FFT
// Expected: prints peak Doppler frequency, surface velocity, and FLOWING/STILL every second
//
// Mount angle matters: adjust MOUNT_ANGLE_RAD to match your physical installation.
// 0.5236 rad = 30 degrees. cos(30) = 0.866.

#include <arduinoFFT.h>

#define ADC_PIN          27
#define SAMPLES          512      // FFT input size — must be power of 2
#define SAMPLE_FREQ      1000.0   // Hz
#define NOISE_FLOOR_HZ   5.0      // Ignore peaks below this — DC + low-freq noise
#define STILL_THRESHOLD  0.05     // m/s — below this = STILL
#define MOUNT_ANGLE_RAD  0.5236   // 30 degrees — adjust per physical installation

double vReal[SAMPLES];
double vImag[SAMPLES];
ArduinoFFT<double> FFT = ArduinoFFT<double>(vReal, vImag, SAMPLES, SAMPLE_FREQ);

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);
  analogSetAttenuation(ADC_11db);
  pinMode(ADC_PIN, INPUT);
  Serial.println("=== HB100 Doppler Radar Velocity Test ===");
  Serial.println("Aim at water surface at 30 degrees along flow direction.");
  Serial.println();
}

void loop() {
  // Collect 512 samples at 1kHz using delayMicroseconds for consistent timing
  for (int i = 0; i < SAMPLES; i++) {
    vReal[i] = analogRead(ADC_PIN) - 2048.0;  // Center around zero
    vImag[i] = 0.0;
    delayMicroseconds(1000);  // 1ms = 1kHz sample rate
  }

  FFT.windowing(FFTWindow::Hamming, FFTDirection::Forward);
  FFT.compute(FFTDirection::Forward);
  FFT.complexToMagnitude();

  // Find peak frequency above noise floor (skip DC bin 0)
  double peakHz = 0.0;
  double peakMag = 0.0;
  int startBin = (int)(NOISE_FLOOR_HZ * SAMPLES / SAMPLE_FREQ) + 1;
  for (int i = startBin; i < SAMPLES / 2; i++) {
    if (vReal[i] > peakMag) {
      peakMag = vReal[i];
      peakHz = (i * SAMPLE_FREQ) / SAMPLES;
    }
  }

  // V = Fd / (72 * cos(theta))
  // 72 Hz/(m/s) is the HB100 Doppler constant for direct radial motion
  // Divide by cos(mount angle) to correct for angled mounting
  float velocityMs = (float)(peakHz / 72.0 / cos(MOUNT_ANGLE_RAD));

  Serial.print("Peak: ");
  Serial.print(peakHz, 1);
  Serial.print(" Hz  Velocity: ");
  Serial.print(velocityMs, 3);
  Serial.print(" m/s  ");
  Serial.println(velocityMs >= STILL_THRESHOLD ? "FLOWING" : "STILL");

  delay(1000);
}
```

- **Step 2: Simulate in Wokwi (limited)**

Wokwi has no HB100 component. Use a potentiometer on GPIO 27 to produce varying ADC levels:

1. Wire a potentiometer to GPIO 27
2. Run the sketch — verify FFT runs without crashing and output prints every second
3. Varying the pot will not produce realistic Doppler peaks; real validation requires hardware over water

- **Step 3: Upload and verify**

1. Assemble the LM358 amplifier circuit on breadboard
2. Upload sketch, open Serial Monitor at 115200
3. With no water / sensor facing a wall: should show near-zero Hz and STILL
4. Wave hand slowly in front of sensor face: should show nonzero Hz and a velocity reading
5. Aim at flowing water in a bucket or basin at 30° along flow: should show consistent nonzero reading and FLOWING

If readings are noisy or erratic: check amplifier wiring, ensure good GND connection between HB100, LM358, and Lolin32.

- **Step 4: Calibration note**

The 72 Hz/(m/s) constant assumes direct radial (head-on) movement. At 30° mount angle, dividing by cos(30°) = 0.866 corrects for the angle. To validate against reality:

1. Float a visible object down a known canal length (e.g., 5 m)
2. Time how long it takes in seconds
3. Velocity estimate = 5 / seconds
4. Compare to sensor reading — if consistently off, note a correction factor to carry into `sensors.cpp`

- **Step 5: Commit**

```bash
git add firmware/test_sketches/test_hb100_velocity/
git commit -m "test: add HB100 Doppler radar ADC+FFT surface velocity test sketch"
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
- **Step 1: Create the test sketch**

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

- **Step 2: Simulate in Wokwi (limited)**

Use a pushbutton on GPIO 25 to proxy the KY-003 hall sensor:

1. In Wokwi, add a pushbutton between GPIO 25 and GND
2. Run the sketch — press to simulate bucket tips
3. Verify tip count increments and debounce rejects rapid presses
4. Tipping bucket mechanism test is hardware-only

- **Step 3: Upload and verify**

1. Upload sketch, open Serial Monitor at 115200
2. Wave a magnet near the KY-003 sensor
3. Each pass should increment the tip count by 1
4. Verify debounce works: fast magnet swipes don't double-count

- **Step 3: Tipping bucket mechanism test**

If you've built the DIY tipping bucket:

1. Mount KY-003 next to the pivot point, magnet on the bucket arm
2. Pour measured water into the funnel
3. Count tips vs expected for your bucket volume
4. Adjust `MM_PER_TIP` if needed

- **Step 4: Commit**

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
- **Step 1: Create the test sketch**

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

- **Step 2: Simulate in Wokwi (full)**

This task is fully simulatable:

1. Create Wokwi project: ESP32 + red LED (GPIO 33) + yellow LED (GPIO 12) + blue LED (GPIO 13) + buzzer (GPIO 32)
2. Run the sketch — watch LEDs cycle through all 5 patterns automatically
3. Verify: NORMAL = all off, YELLOW = yellow only, ORANGE = red + yellow, RED = red only, CLOG = blue only
4. Verify timing: each pattern runs ~5 seconds with 2-second pause between
5. Save `diagram.json` to `wokwi/task04_buzzer_leds/`

- **Step 3: Upload and verify**

1. Upload sketch, open Serial Monitor at 115200
2. Watch and listen as it cycles through each pattern:
  - NORMAL: silent, all LEDs off
  - YELLOW: short beep repeating, yellow LED on
  - ORANGE: fast double beep, red + yellow LEDs on
  - RED: continuous tone, red LED on
  - CLOG: slow triple beep, blue LED on
3. Verify each pattern is distinguishable by sound alone (field crew won't be looking at LEDs)

- **Step 3: Commit**

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
- **Step 1: Create the test sketch**

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

- **Step 2: Simulate in Wokwi (full)**

Fully simulatable using a potentiometer:

1. In Wokwi, add a potentiometer: wiper → GPIO 35, one end → 3.3V, other → GND
2. Run the sketch — adjust the pot to simulate different voltage levels
3. Verify: percentage calculates correctly, LOW BATTERY flag triggers below 3300 mV, CRITICAL below 3000 mV
4. Verify 16-sample averaging produces stable readings
5. Save `diagram.json` to `wokwi/task05_battery_adc/`

- **Step 3: Upload and verify**

1. Upload sketch, open Serial Monitor at 115200
2. With USB power only (no battery): reading will be 0 or erratic — that's expected
3. Connect a charged 18650: should read 3700–4200 mV
4. Compare reading against a multimeter on the battery terminals
5. If readings are off by more than 100mV, adjust `DIVIDER_RATIO`

- **Step 3: Commit**

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

- **Step 1: Install RadioLib library**

In Arduino IDE: Sketch → Include Library → Manage Libraries → search "RadioLib" → Install

- **Step 2: Create the TX test sketch**

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

- **Step 2b: Simulate in Wokwi (stub only)**

Wokwi does not support SX1278/Ra-02. To validate packet formatting and timing:

1. Comment out `#include <RadioLib.h>` and all RadioLib calls
2. Replace `radio.begin(...)` with `Serial.println("LoRa init stubbed");`
3. Replace `radio.transmit(msg)` with `Serial.println(msg)`
4. Run in Wokwi — verify packet numbering increments and 3-second interval is correct
5. Restore original code before uploading to hardware

RF communication requires physical boards.

- **Step 3: Upload to board #1**

1. Connect Lolin32 #1 (with Ra-02 + antenna attached)
2. Upload sketch
3. Open Serial Monitor — should print "Initializing LoRa... OK" then "Sending..." every 3s
4. If init fails: double-check SPI wiring, especially NSS (GPIO 5) and RST (GPIO 14)

- **Step 4: Commit**

```bash
git add firmware/test_sketches/test_lora_tx/
git commit -m "test: add LoRa point-to-point TX test sketch"
```

---

### Task 7: LoRa Point-to-Point RX

**Files:**

- Create: `firmware/test_sketches/test_lora_rx/test_lora_rx.ino`

**What you need:** Lolin32 #2, Ra-02 module, 12dBi antenna, same wiring as Task 6

- **Step 1: Create the RX test sketch**

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

- **Step 1b: Simulate in Wokwi (stub only)**

Same stub approach as Task 6:

1. Comment out RadioLib and replace `radio.receive(...)` with a hardcoded test string
2. Verify Serial output formatting for received packet, RSSI, and SNR display
3. Restore original code before uploading to hardware

- **Step 2: Upload to board #2 and verify both boards**

1. Upload to Lolin32 #2 (with Ra-02 + antenna)
2. Open Serial Monitor on both boards (use two Arduino IDE windows or two COM ports)
3. Board #1 (TX) should print "Sending..." every 3s
4. Board #2 (RX) should print "Received: FloodWatch TX test #N" with RSSI/SNR values
5. RSSI closer to 0 = stronger signal. Typical indoor: -30 to -80 dBm

- **Step 3: Range test**

1. Move boards apart — different rooms, different floors
2. Note RSSI at each distance
3. Verify packets still arrive through walls (433MHz should penetrate well)
4. Find the point where packets start dropping

- **Step 4: Commit**

```bash
git add firmware/test_sketches/test_lora_rx/
git commit -m "test: add LoRa point-to-point RX test sketch"
```

---

## Layer 3–6 — Production Firmware

### Task 8: config.h — Pin Definitions and Constants

**Files:**

- Create: `firmware/FloodWatch_Node/config.h`
- **Step 1: Create config.h**

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
#define PIN_FLOW        27   // HB100 amplified IF output (ADC)
#define PIN_RAIN        25   // KY-003 signal (interrupt)

// ── HB100 Doppler constants ───────────────────────────────
#define MOUNT_ANGLE_RAD              0.5236   // 30 degrees — adjust per physical installation
#define VELOCITY_NOISE_THRESHOLD_HZ  5.0      // Ignore FFT peaks below this

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
  uint8_t  node_id;          // From chip ID (last byte of ESP32 MAC)
  uint8_t  packet_type;      // 0x01 = sensor, 0x02 = heartbeat
  uint16_t water_level_cm;   // JSN-SR04T reading
  uint16_t surface_velocity_mmps;  // HB100 surface velocity in mm/s (0 if absent)
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

- **Step 2: Commit**

```bash
git add firmware/FloodWatch_Node/config.h
git commit -m "feat: add config.h with pin definitions and packet format"
```

> **Wokwi:** No standalone simulation for this task. Validated as part of Task 14 simulation.

---

### Task 9: detect module — Auto-Detection and Chip ID

**Files:**

- Create: `firmware/FloodWatch_Node/detect.h`
- Create: `firmware/FloodWatch_Node/detect.cpp`
- **Step 1: Create detect.h**

```cpp
// detect.h
#ifndef FLOODWATCH_DETECT_H
#define FLOODWATCH_DETECT_H

#include "config.h"

// Reads chip ID and probes sensors to populate NodeConfig.
// Call once in setup() before initializing any other module.
NodeConfig detectHardware();

// Print detected config to Serial for debugging
void printNodeConfig(const NodeConfig& cfg);

#endif
```

- **Step 2: Create detect.cpp**

```cpp
// detect.cpp
#include "detect.h"

static uint8_t readChipId() {
  // Use the last byte of the ESP32's unique MAC address as node ID.
  // Guaranteed unique per chip — no hardware or provisioning needed.
  return (uint8_t)(ESP.getEfuseMac() & 0xFF);
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
  // HB100: when present and powered, the amplified IF output shows noise variance
  // above the ADC noise floor. When absent, GPIO 27 floats near zero with minimal variance.
  // Sample ADC for 100ms — if variance exceeds noise threshold, sensor is active.
  analogReadResolution(12);
  delay(10);

  int samples[20];
  for (int i = 0; i < 20; i++) {
    samples[i] = analogRead(PIN_FLOW);
    delay(5);
  }

  int minVal = samples[0], maxVal = samples[0];
  for (int i = 1; i < 20; i++) {
    if (samples[i] < minVal) minVal = samples[i];
    if (samples[i] > maxVal) maxVal = samples[i];
  }

  return (maxVal - minVal) > 50;  // Variance > 50 ADC counts = amplifier circuit active
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
  cfg.node_id  = readChipId();
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

- **Step 3: Create a minimal .ino to test detection**

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

- **Step 4: Upload and verify detection**

1. Upload with NO sensors plugged in → all should show "not connected"
2. Plug in JSN-SR04T → re-scan should detect it
3. Connect HB100 amplifier circuit → should detect flow sensor (ADC variance check)
4. Verify node ID printed to Serial matches last byte of the chip's MAC
5. Verify detection is reliable (run 5+ scans, no false positives)

- **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/config.h firmware/FloodWatch_Node/detect.h firmware/FloodWatch_Node/detect.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add hardware auto-detection and chip ID reading"
```

> **Wokwi:** No standalone simulation for this task. Validated as part of Task 14 simulation.

---

### Task 10: sensors module

**Files:**

- Create: `firmware/FloodWatch_Node/sensors.h`
- Create: `firmware/FloodWatch_Node/sensors.cpp`
- **Step 1: Create sensors.h**

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

- **Step 2: Create sensors.cpp**

```cpp
// sensors.cpp
#include "sensors.h"

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
static uint32_t _lastRainTips = 0;

void sensorsInit(const NodeConfig& cfg) {
  // JSN-SR04T is always expected (core sensor)
  Serial2.begin(9600, SERIAL_8N1, PIN_JSN_RX, PIN_JSN_TX);

  if (cfg.has_flow) {
    // HB100: ADC input, no interrupt — sampled on demand via readSurfaceVelocityMmps()
    analogSetAttenuation(ADC_11db);
    pinMode(PIN_FLOW, INPUT);
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

static uint16_t readSurfaceVelocityMmps() {
  // Sample HB100 amplified IF output via ADC, apply FFT to extract dominant frequency.
  // V = Fd / (72 * cos(MOUNT_ANGLE_RAD))
  // Returns surface velocity in mm/s (multiply m/s by 1000).
  const int SAMPLES = 512;
  const float SAMPLE_FREQ = 1000.0;
  double vReal[SAMPLES], vImag[SAMPLES];

  for (int i = 0; i < SAMPLES; i++) {
    vReal[i] = analogRead(PIN_FLOW) - 2048.0;
    vImag[i] = 0.0;
    delayMicroseconds(1000);  // 1ms = 1kHz sample rate
  }

  ArduinoFFT<double> fft(vReal, vImag, SAMPLES, SAMPLE_FREQ);
  fft.windowing(FFTWindow::Hamming, FFTDirection::Forward);
  fft.compute(FFTDirection::Forward);
  fft.complexToMagnitude();

  // Find peak above noise floor (skip DC bin 0)
  double peakHz = 0.0;
  double peakMag = 0.0;
  int startBin = (int)(VELOCITY_NOISE_THRESHOLD_HZ * SAMPLES / SAMPLE_FREQ) + 1;
  for (int i = startBin; i < SAMPLES / 2; i++) {
    if (vReal[i] > peakMag) {
      peakMag = vReal[i];
      peakHz = (i * SAMPLE_FREQ) / SAMPLES;
    }
  }

  if (peakHz < VELOCITY_NOISE_THRESHOLD_HZ) return 0;
  float velocityMs = (float)(peakHz / 72.0 / cos(MOUNT_ANGLE_RAD));
  return (uint16_t)(velocityMs * 1000.0f);  // mm/s
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
  pkt.surface_velocity_mmps = cfg.has_flow ? readSurfaceVelocityMmps() : 0;
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
  _rainTipCount = 0;
  interrupts();
}
```

- **Step 3: Update .ino to test sensor readings**

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
  Serial.print("  Velocity:    "); Serial.print(pkt.surface_velocity_mmps / 1000.0, 3); Serial.println(" m/s");
  Serial.print("  Rain tips:   "); Serial.println(pkt.rain_tips);
  Serial.print("  Battery:     "); Serial.print(pkt.battery_mv); Serial.println(" mV");
  Serial.print("  Flags:       0x"); Serial.println(pkt.flags, HEX);
  Serial.println();

  delay(5000);
}
```

- **Step 4: Upload and verify**

1. Upload with JSN-SR04T connected → water level should show valid cm reading
2. If flow/rain sensors connected → those fields should populate
3. Absent sensors should show 0 with flags unset
4. Battery reading should be reasonable

- **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/sensors.h firmware/FloodWatch_Node/sensors.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add sensors module with water level, flow, rain, battery"
```

> **Wokwi:** No standalone simulation for this task. Validated as part of Task 14 simulation.

---

### Task 11: actuators module

**Files:**

- Create: `firmware/FloodWatch_Node/actuators.h`
- Create: `firmware/FloodWatch_Node/actuators.cpp`
- **Step 1: Create actuators.h**

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

- **Step 2: Create actuators.cpp**

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

- **Step 3: Test by adding to .ino**

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

- **Step 4: Upload and verify**

1. Upload, open Serial Monitor
2. Type `0` through `4` to test each alert pattern
3. Verify buzzer patterns match spec, LEDs light correct colors
4. Verify `actuatorsUpdate()` is non-blocking (sensor readings still print)

- **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/actuators.h firmware/FloodWatch_Node/actuators.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add actuators module with non-blocking buzzer patterns"
```

> **Wokwi:** No standalone simulation for this task. Actuator patterns are validated in Task 4 simulation and the full state machine in Task 14 simulation.

---

### Task 12: lora_comm module — LoRaMesher Integration

**Files:**

- Create: `firmware/FloodWatch_Node/lora_comm.h`
- Create: `firmware/FloodWatch_Node/lora_comm.cpp`

**Prerequisites:** Install LoRaMesher library in Arduino IDE. Check [https://github.com/LoRaMesher/LoRaMesher](https://github.com/LoRaMesher/LoRaMesher) for installation instructions and current API. The code below follows the documented API pattern — verify against the version you install.

- **Step 1: Create lora_comm.h**

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

- **Step 2: Create lora_comm.cpp**

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

- **Step 3: Test with two boards**

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

- **Step 4: Upload to both boards and verify mesh**

1. Upload to both Lolin32 boards (both with Ra-02 + antenna)
2. Both should show "LoRa mesh active" with different local addresses
3. Each board should send packets every 30 seconds
4. Open Serial Monitor on both — verify packets are being sent
5. At this point, without a base station, you won't see received data, but LoRaMesher should log routing table updates

- **Step 5: Commit**

```bash
git add firmware/FloodWatch_Node/lora_comm.h firmware/FloodWatch_Node/lora_comm.cpp firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: add LoRaMesher communication module"
```

> **Wokwi:** LoRaMesher cannot be simulated. This module is validated hardware-only in Tasks 6–7 (point-to-point LoRa) and Task 15 (mesh integration).

---

### Task 13: power module — Deep Sleep and Transistor Switching

**Files:**

- Create: `firmware/FloodWatch_Node/power.h`
- Create: `firmware/FloodWatch_Node/power.cpp`
- **Step 1: Create power.h**

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

- **Step 2: Create power.cpp**

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

- **Step 3: Commit**

```bash
git add firmware/FloodWatch_Node/power.h firmware/FloodWatch_Node/power.cpp
git commit -m "feat: add power module with deep sleep and transistor switching"
```

> **Wokwi:** Deep sleep timer wake and transistor switching (GPIO 4) can be tested in Wokwi. Use an LED on GPIO 4 as a proxy for the transistor gate. Validated fully as part of Task 14 simulation.

---

### Task 14: Main Firmware — FloodWatch_Node.ino (Final)

**Files:**

- Modify: `firmware/FloodWatch_Node/FloodWatch_Node.ino`
- **Step 1: Write the final main sketch**

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
  Serial.print("cm vel=");
  Serial.print(pkt.surface_velocity_mmps);
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

- **Step 2: Simulate in Wokwi (composite)**

Simulate the full wake cycle with stubs for unsupported components:

1. Create Wokwi project: ESP32 + red LED (GPIO 33) + yellow LED (GPIO 12) + blue LED (GPIO 13) + buzzer (GPIO 32) + potentiometer (GPIO 35) + pushbutton for flow (GPIO 27) + pushbutton for rain (GPIO 25)
2. Stub `lora_comm`:
  - `loraInit()` → `return true`
  - `loraSendUplink()` → `Serial.println("uplink sent (stubbed)")`
  - `loraReceiveDownlink()` → return a hardcoded alert state after N cycles to test alert path
3. Stub `detectJsnSr04t()` in `detect.cpp` → `return true` (always detected)
4. Stub `readWaterLevelCm()` → `return 120`
5. Run in Wokwi and verify:
  - Boot prints correct node config and chip ID
  - Sensor packet is formatted and "sent" each cycle
  - Deep sleep triggers after no alert
  - When stub returns an alert downlink: node stays awake, correct LED/buzzer pattern activates
6. Save `diagram.json` to `wokwi/task14_main_firmware/`

This catches state machine bugs (wrong pin, bad sleep logic, alert not clearing) before touching hardware.

- **Step 3: Upload and verify full cycle**

1. Upload to a Lolin32 with Ra-02 + JSN-SR04T connected
2. Open Serial Monitor — should see detection, sensor reading, uplink sent
3. If no alert received → should print "Entering deep sleep..." and go quiet
4. After 30 seconds → should wake and repeat
5. Verify power consumption: during sleep, current should drop to microamps (if you have a multimeter in series with battery)

- **Step 3: Test alert override**

Without a base station sending real downlinks, test the alert-stays-awake behavior:

1. Temporarily modify the code to simulate receiving an alert (hardcode `currentAlert = ALERT_YELLOW` after the RX window)
2. Verify the node stays awake, runs the buzzer pattern, and continues reading sensors
3. Remove the hardcoded alert, re-upload

- **Step 4: Commit**

```bash
git add firmware/FloodWatch_Node/FloodWatch_Node.ino
git commit -m "feat: complete main firmware with sleep/wake cycle and alert override"
```

---

### Task 15: Two-Node Integration Test

> **Wokwi:** Not applicable. LoRa mesh communication requires two physical boards. All simulatable logic has been validated in prior tasks.

**Files:** No new files — this is a verification task.

**What you need:** Both Lolin32 boards fully wired (Ra-02 + antenna + JSN-SR04T minimum).

- **Step 1: Set up Node A**

1. Connect JSN-SR04T + HB100 with LM358 amplifier circuit + rain gauge (if available)
2. Upload `FloodWatch_Node` firmware
3. Open Serial Monitor — verify node ID printed, detection shows all connected sensors

- **Step 2: Set up Node B**

1. Connect JSN-SR04T only
2. Upload same `FloodWatch_Node` firmware
3. Open Serial Monitor — verify node ID is different from Node A, detection shows JSN-SR04T only, flow/rain absent

- **Step 3: Verify LoRa mesh communication**

1. Power both nodes simultaneously
2. Both should initialize LoRaMesher and show mesh addresses
3. Wait for routing table updates in Serial output (LoRaMesher logs these)
4. Both nodes should be sending uplink packets every 30 seconds
5. Verify both nodes appear in each other's routing tables

- **Step 4: Verify deep sleep cycle**

1. Watch both nodes cycle: wake → read → send → sleep → wake
2. Verify the 30-second interval is consistent
3. If you have a multimeter: measure current during sleep (should be ~10µA)

- **Step 5: Document results**

Create a simple test log noting:

- Node IDs detected correctly? (yes/no)
- Sensors auto-detected correctly? (yes/no)
- LoRa packets sent/received? (yes/no, RSSI values)
- Deep sleep working? (yes/no, measured current if available)
- Any issues encountered and how they were resolved
- **Step 6: Commit test log**

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