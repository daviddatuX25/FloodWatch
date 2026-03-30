# Adaptive Sleep Mode
**Status:** Spec approved, pending implementation (Layer 5)
**Affects:** All field nodes, base station sub-project

---

## Overview

FloodWatch nodes normally sleep 30s between measurements (ACTIVE mode). Adaptive sleep lets the base station command nodes into lower-power modes during clear weather, significantly extending battery life. The HB100 Doppler radar (40mA @ 5V) is the primary power win — it is skipped in all non-ACTIVE modes.

JSN-SR04T water level monitoring runs in **all modes** — the water is always being watched.

---

## Three Modes

| Mode | Constant | Sleep Cycle | HB100 | JSN-SR04T | Rain | Runtime on 2200mAh |
|------|----------|-------------|-------|-----------|------|-------------------|
| ACTIVE | `0` | 30s | YES | YES | YES | ~200 hours |
| REDUCED | `1` | 5 min | NO | YES | YES | ~1,700 hours |
| WATCHDOG | `2` | 30 min | NO | YES | YES | ~8,800 hours |

### Override Rules (always win)

1. **Alert override** — if `alert_state != NORMAL`, node ignores `sleep_mode` and behaves as ACTIVE for that cycle. Flood is happening; battery saving is irrelevant.
2. **Daily health check** — in WATCHDOG mode, if 24 hours pass without a successful LoRa uplink, node forces one 30s sync wakeup then returns to WATCHDOG.

---

## Hardware Requirement

Add one 2N2222A transistor + 1kΩ resistor to core PCB to switch HB100 power:

```
GPIO 21 ──[1kΩ]── 2N2222A base
                  collector ── HB100 VCC (5V)
                  emitter  ── GND
```

GPIO 21 is free (no I2C devices in design). The Lolin32 board has a 100kΩ pull-up on GPIO 21 — harmless. GPIOs 34/36 are input-only on ESP32 and cannot be used here.

**Power group (all transistor switches):**

| GPIO | Switches |
|------|---------|
| 4 | Ra-02 LoRa module |
| 21 | HB100 Doppler radar |
| 32 | Buzzer |

---

## Packet Protocol Changes

### Downlink v2 — 3 bytes

| Byte | Field | Values |
|------|-------|--------|
| 0 | `target_node_id` | 0xFF = broadcast |
| 1 | `alert_state` | 0=normal 1=yellow 2=orange 3=red 4=clog |
| 2 | `sleep_mode` | 0=ACTIVE 1=REDUCED 2=WATCHDOG |

If a 2-byte packet is received (old base station format), only `alert_state` is updated; `sleep_mode` stays unchanged.

### Uplink — still 10 bytes

`flags` byte gains two bits for current mode (no size change):

| Bit(s) | Field |
|--------|-------|
| 0 | `has_flow` |
| 1 | `has_rain` |
| 2 | `low_battery` |
| 3–4 | `current_mode` (0b00=ACTIVE, 0b01=REDUCED, 0b10=WATCHDOG) |
| 5–7 | reserved |

---

## RTC Memory

Survives deep sleep on ESP32 RTC slow memory:

```cpp
RTC_DATA_ATTR uint8_t  g_sleep_mode    = SLEEP_MODE_ACTIVE; // default safe
RTC_DATA_ATTR uint8_t  g_alert_state   = ALERT_NORMAL;
RTC_DATA_ATTR uint32_t g_last_sync_s   = 0;   // unix time of last LoRa uplink
RTC_DATA_ATTR uint8_t  g_rain_tips_rtc = 0;   // accumulated tips across long sleeps
```

> `g_rain_tips_rtc` is a `uint8_t` — overflows at 255 tips (= 51mm in 30min). Consider upgrading to `uint16_t` if deploying during typhoon season.

---

## Firmware Loop Changes

Key additions to `FloodWatch_Node.ino`:

```
on wake:
  effective_mode = g_sleep_mode
  if g_alert_state != NORMAL → effective_mode = ACTIVE

  power on Ra-02 (GPIO 4 HIGH)
  if effective_mode == ACTIVE AND has_flow:
      GPIO 21 HIGH (HB100 on)
      delay 200ms  ← warmup required

  read sensors per effective_mode
  GPIO 21 LOW (HB100 off, before LoRa to reduce interference)

  build + send uplink (flags bits 3-4 = current_mode)
  listen 1s for downlink
    3-byte: update g_alert_state + g_sleep_mode + g_last_sync_s
    2-byte: update g_alert_state only

  power off Ra-02 (GPIO 4 LOW)

  if g_alert_state != NORMAL → stay awake (buzzer/LED loop)
  else:
    ACTIVE:   sleep 30s
    REDUCED:  sleep 5min
    WATCHDOG: if now - g_last_sync_s > 86400 → sleep 30s (force sync)
              else → sleep 30min
```

---

## Base Station Weather Logic

Base station (Orange Pi, Python) fetches OpenWeatherMap 3-hour forecast every 15 minutes and decides mode:

```python
def decide_sleep_mode(forecast_slots, current_alert_state) -> int:
    next_3h  = forecast_slots[0]
    next_24h = forecast_slots[:8]   # 24h = 8 × 3h slots
    next_48h = forecast_slots[:16]

    if current_alert_state >= ALERT_YELLOW:
        return SLEEP_MODE_ACTIVE

    if next_3h.pop_pct >= 30 or next_3h.rain_mm > 0.5:
        return SLEEP_MODE_ACTIVE

    if any(slot.pop_pct >= 20 for slot in next_24h):
        return SLEEP_MODE_REDUCED

    if all(slot.pop_pct < 20 and slot.rain_mm < 0.1 for slot in next_48h):
        return SLEEP_MODE_WATCHDOG

    return SLEEP_MODE_REDUCED  # conservative default
```

Tunable thresholds in base station config:
- `POP_ACTIVE_THRESHOLD = 30`
- `POP_REDUCED_THRESHOLD = 20`

OpenWeatherMap free tier: ~96 calls/day at 15min polling, well within 1000/day limit.

---

## Files to Change

| File | Change |
|------|--------|
| `firmware/FloodWatch_Node/config.h` | Add `SLEEP_MODE_*` constants, `HB100_PWR_PIN 21`, sleep durations |
| `firmware/FloodWatch_Node/FloodWatch_Node.ino` | `RTC_DATA_ATTR` vars, mode logic, health check, alert override |
| `firmware/FloodWatch_Node/power.cpp/.h` | `hb100PowerOn()`, `hb100PowerOff()`, `getSleepDurationS(mode)` |
| `firmware/FloodWatch_Node/lora_comm.cpp/.h` | Handle 3-byte downlink v2 |
| `firmware/FloodWatch_Node/sensors.cpp` | Guard HB100 read behind `effective_mode == ACTIVE` |
| `firmware/test_sketches/test_adaptive_sleep/` | Test sketch for mode transitions + RTC persistence |

---

## Implementation Tasks (Layer 5)

### Task 13b — HB100 Power Switch (Hardware)
Wire GPIO 21 → 1kΩ → 2N2222A → HB100 VCC. Smoke test: GPIO 21 HIGH/LOW, verify IF signal on ADC (GPIO 27) appears/disappears.

### Task 13c — Adaptive Sleep Test Sketch
`test_adaptive_sleep.ino`: Serial input `A`/`R`/`W` sets mode, node sleeps correct duration, prints mode + GPIO 21 state on wake. Verify RTC survives hard reset.

### Task 13d — Downlink v2 Integration
Send `[0xFF, 0x00, 0x02]` → WATCHDOG. Send `[0xFF, 0x01, 0x02]` → alert overrides to ACTIVE, buzzes. Send `[0xFF, 0x00, 0x00]` → returns to ACTIVE 30s cycle.

---

## Mesh Relay Constraint ⚠️

**Deep sleep breaks LoRaMesher relay.** A node in WATCHDOG (30min sleep) or REDUCED (5min sleep) has its Ra-02 powered off and cannot forward packets from other nodes. In a multi-hop chain, a sleeping intermediate node silently isolates all nodes behind it.

**Current constraint (prototype):** REDUCED and WATCHDOG modes are only safe when **every node has direct LoRa range to the base station**. Do not assign deep sleep modes to relay-critical intermediate nodes.

**For multi-hop deployments (future):** Decouple sensor duty cycle from relay duty cycle:
- Relay wakeup: every ~2 minutes (brief Ra-02 listen window, ~2s)
- Sensor wakeup: per sleep mode (30s / 5min / 30min)
- Ra-02 stays in intermittent listen mode between sensor reads
- Significant firmware complexity increase — defer to Layer 5 design review

**Open item:** Before deploying beyond 2-node prototype, audit mesh topology and mark which nodes are relay-critical. Relay-critical nodes must stay at ACTIVE or use relay-aware sleep.

---

## Open Items

- [ ] GPIO 21 bench test: confirm 100kΩ pull-up does not prevent transistor from fully turning off
- [ ] GPIO 22 — reserve for future I2C (BME680 humidity sensor option)
- [ ] Rain tip counter overflow: `uint8_t` → consider `uint16_t` for typhoon conditions
- [ ] Firebase dashboard: add `sleep_mode` field to node status view
- [ ] Base station Python cron job (separate sub-project — this spec defines the interface)
- [ ] Multi-hop relay-aware sleep: design relay wakeup interval separate from sensor interval (defer to Layer 5)
