# Smart Chicken Coop Monitor — IoT Device Project Plan

> **Project Type:** Custom Hardware IoT Device  
> **Integration Target:** Home Assistant  
> **Solo Developer:** Yes — design, hardware, and firmware by one person  
> **Power:** Mains (wall power, always on)  
> **Connectivity:** Wi-Fi + Wired peripherals (I2C, UART, SPI)  
> **Status:** Redesign of existing device

---

## Table of Contents

1. [Project Vision & Goals](#1-project-vision--goals)
2. [Feature Scope](#2-feature-scope)
3. [Hardware Architecture Recommendation](#3-hardware-architecture-recommendation)
4. [Firmware & Integration Stack](#4-firmware--integration-stack)
5. [Phase Breakdown & Checkpoints](#5-phase-breakdown--checkpoints)
6. [Risk Register](#6-risk-register)
7. [v2 Backlog](#7-v2-backlog)
8. [Decisions Log](#8-decisions-log)

---

## 1. Project Vision & Goals

Design and build a second-generation smart Chicken Coop feeder and monitoring station that replaces an existing device. The new device will be reliable, locally controlled via Home Assistant, and expandable for future features.

### Design Principles

- **Reliability first** — Door must never silently fail
- **Local-first** — fully functional without cloud; HA is the hub
- **Maintainability** — clean wiring, documented firmware, easy to reflash
- **Expandable** — hardware and firmware designed with v2 features in mind
- **Solo-scale** — no over-engineering; keep scope tightly managed per phase

### Pain Points Being Solved (from existing device)

- No unified control system. multiple scripts, each one ran it's own hardware monitor.
- used Cron jobs to coordinate script runs.
- all data was recoded in a local DB, a local web server rendered ASP pages.
- had no HA componants

---

## 2. Feature Scope

### v1 — Must Have

| # | Feature | Notes |
|---|---------|-------|
| F1 | **Scheduled Door** | Time-based, configurable from HA |
| F2 | **Remote manual Door** | Trigger from HA dashboard or mobile |
| F3 | **Food level alert** | Notify when hopper is low |
| F4 | **Water level alert** | Notify when bowl/reservoir is low |
| F5 | **Local display** | Show todays egg count |
| F6 | **Local input controls** | 4x4 keypad, only local controll will be egg count |
| F7 | **Motion / presence detection** | Know when a chicken is in next box |
| F8 | **Temperature & humidity monitoring** | Environmental data reported to HA |
| F9 | **Status LEDs** | Visual indicators for power, feed, alert states |
| F10 | **Stepper mottor Door control** | Auger/disc mechanism to portion food |
| F11 | **Relay output** | Switch a connected load (e.g., water pump, fountain) |
| F12 | **Wi-Fi OTA firmware updates** | Reflash without physical access |
| F13 | **Power loss recovery** | Restore state correctly on reboot |

### v2 — Future (Scoped Out of v1)

- Live camera feed (MJPEG stream to HA)
- Power/energy monitoring of connected loads
- Pet recognition via camera (ML)
- Food weight measurement (load cell)
- Automatic water top-up

---

## 3. Hardware Architecture Recommendation

### 3.1 Microcontroller — ESP32-S3

**Recommended MCU: Espressif ESP32-S3 (e.g., ESP32-S3-DevKitC-1)**

| Criteria | Rationale |
|----------|-----------|
| Wi-Fi | Built-in 2.4 GHz 802.11 b/g/n — no external module needed |
| GPIO count | 45 usable GPIOs — enough for all peripherals |
| I2C / SPI / UART | Hardware-native buses, multiple instances |
| Camera interface | Built-in DVP/MIPI CSI for future camera (v2) |
| ESPHome support | First-class support — ideal for HA integration |
| PSRAM | Up to 8 MB PSRAM available — handles display framebuffer and future camera |
| USB native | USB OTG for easier development and flashing |
| Power | 3.3V logic; pair with a 5V → 3.3V LDO |
| Cost | ~$5–10 USD in module form |

> **Alternative considered:** Raspberry Pi Zero 2W — rejected for v1 due to boot time, SD card reliability, and complexity overkill for the sensor/actuator mix. Revisit if camera AI (v2) demands it.

---

### 3.2 Peripheral Component Selections

#### Sensors & Inputs

| Component | Part Suggestion | Bus | Notes |
|-----------|----------------|-----|-------|
| Temp / Humidity | SHT31 or AHT21 | I2C | More accurate and stable than DHT22 |
| Food level | JSN-SR04T ultrasonic or VL53L0X ToF | UART / I2C | Contactless, inside hopper |
| Water level | Capacitive water level sensor (XKC-Y25) | GPIO (digital) | No exposed electrodes — safer around water |
| Motion / Presence | HC-SR501 PIR or LD2410 mmWave | GPIO / UART | mmWave detects still pets; PIR is simpler |
| Rotary encoder | EC11 encoder with push button | GPIO | Local UI navigation |
| 4x4 keypad | Tactile push buttons | GPIO | enter and reset daily egg count |
| Extra buttons | Tactile push buttons | GPIO | Manual feed button, reset |

#### Outputs & Actuators

| Component | Part Suggestion | Interface | Notes |
|-----------|----------------|-----------|-------|

| 2x 7 segment Display |  | GPIO | Simple, 2 digit display, minimalist |
| Stepper motor | NEMA 17 or 28BYJ-48 | STEP/DIR GPIO | Auger drive; NEMA 17 for more torque |
| Stepper driver | A4988 or TMC2208 | STEP/DIR/EN | TMC2208 = silent operation |
| Relay | Single or dual 5V relay module | GPIO | Controls water pump or fountain |
| LED strip | WS2812B addressable LEDs | GPIO (1-wire) | Status indicators, ambient lighting |
| Status LED | Discrete RGB LED or NeoPixel | GPIO | Power/feed/alert states |
| Buzzer (optional) | Passive piezo buzzer | GPIO (PWM) | Optional audio alerts |
| Display (optional, v2 maybe)| 2.4"–3.5" ILI9341 TFT (320×240) | SPI | Color, fast refresh, good library support |

#### Power

| Rail | Source | Consumers |
|------|--------|-----------|
| 5V (main) | 5V 3A wall adapter + barrel jack | Relay, stepper driver, display backlight, relay coils |
| 3.3V | AMS1117-3.3 LDO from 5V | ESP32-S3, sensors, encoder, LEDs |
| Motor power | Shared 5V or separate 12V | Depends on stepper motor chosen; NEMA 17 may need 12V |

> **Note:** If using a NEMA 17 stepper, consider a dual-rail supply: 5V for logic and 12V for the motor. A 12V 2A supply with a 5V buck converter covers both rails cleanly.

---

### 3.3 Bus / GPIO Map (Preliminary)

| Bus | Devices |
|-----|---------|
| I2C (Bus 0) | SHT31 (temp/humidity), VL53L0X (food level, optional) |
| SPI | ILI9341 TFT display |
| UART | JSN-SR04T (if ultrasonic food sensor), LD2410 (if mmWave) |
| GPIO (digital in) | PIR/mmWave motion, water level sensor, encoder A/B/SW, buttons |
| GPIO (digital out) | Relay, stepper STEP/DIR/EN, buzzer |
| GPIO (RMT/PWM) | WS2812B LED strip |

> Final GPIO pin assignments are a Phase 4 deliverable.

---

## 4. Firmware & Integration Stack

### 4.1 Firmware Platform — ESPHome

**Recommended: ESPHome (YAML-based, compiled to C++)**

| Criteria | Rationale |
|----------|-----------|
| HA integration | Native ESPHome API — auto-discovery, no MQTT broker needed |
| OTA updates | Built-in, triggered from HA or web UI |
| Sensor support | First-class support for SHT31, ultrasonic, PIR, I2C, SPI |
| Display | Supports ILI9341 via `display:` component with LVGL (ESPHome ≥ 2024.x) |
| Stepper | `stepper:` component built in |
| Extensibility | Custom `lambda:` C++ for anything not natively supported |
| Reliability | Mature, battle-tested, watchdog-protected |

> **Alternative considered:** Custom Arduino C++ firmware with MQTT. More control, but significantly more development time. Revisit only if ESPHome hits hard limitations.

### 4.2 Home Assistant Integration Points

| HA Entity | Type | Source |
|-----------|------|--------|
| `sensor.feeder_temperature` | Sensor | SHT31 |
| `sensor.feeder_humidity` | Sensor | SHT31 |
| `sensor.feeder_feed_level` | Binary Sensor | Food level threshold |
| `binary_sensor.feeder_water_low` | Binary Sensor | Water level sensor |
| `binary_sensor.feeder_motion` | Binary Sensor | PIR / mmWave |
| `button.feeder_door_close` | Button | Triggers manual door close |
| `button.feeder_door_open` | Button | Triggers manual door open |
| `switch.feeder_water_relay` | Switch | Controls relay output |
| `select.feeder_schedule` | Select / Automation | Door schedule |
| `light.feeder_leds` | Light | LED strip control |

### 4.3 Feeding State Machine

```
[IDLE] ──── schedule trigger / HA button ───→ [OPEN/CLOSE]
  ↑                                                 │
  └──────────────── complete ──────────────────────┘
  │
  ├── food low threshold ───→ [ALERT: FOOD LOW]
  ├── water low threshold ──→ [ALERT: WATER LOW]
  └── hardware fault ───────→ [ERROR] → HA notification
```

---

## 5. Phase Breakdown & Checkpoints

---

### ✅ Phase 0 — Project Foundation
**Goal:** Close all open questions. No hardware ordered, no code written.

**Tasks:**
- [ ] Document all pain points of the existing device (list what it fails at)
- [ ] Confirm v1 feature list (lock F1–F13, officially defer v2)
- [x] Confirm MCU choice (ESP32-S3) and firmware platform (ESPHome)
    - ESP32-S3 and ESPHome
- [ ] Confirm stepper motor type and whether 12V is needed
- [x] Confirm display size and type (TFT ILI9341 recommended)
    - initial display should just be the 2 7-segment displays
- [ ] Confirm motion sensor approach (PIR vs mmWave)
- [x] Confirm food level sensor approach (ultrasonic vs ToF)
    - HC-SR04 Ultasonic sensor, available on hand
- [x] Define physical form factor (countertop footprint, rough dimensions)
    - Must fit in a Hoffman A806CH (6"x8" enclosure)
- [x] Define hopper volume target (how much food does it hold?)
    - Hopper volume will be roughly 40lbs, but will be meaused by distance from top

**Checkpoint 0 Exit Criteria:**
> All open hardware and feature decisions are documented and locked. A complete component shortlist exists. No ambiguity on v1 scope.

---

### 🔲 Phase 1 — Component Selection & BOM
**Goal:** A final, ordered Bill of Materials with sources and cost.

**Tasks:**
- [ ] Finalize all component selections (MCU, sensors, motor, display, LEDs, relay)
- [ ] Create full BOM spreadsheet: part name, qty, supplier, unit cost, link
- [ ] Calculate total power budget (mA per rail at worst case)
- [ ] Confirm power supply spec (voltage, amperage, connector type)
- [ ] Identify any long-lead-time parts and order early
- [ ] Order all components

**Checkpoint 1 Exit Criteria:**
> BOM is complete. All parts ordered. Power budget validates the supply choice.

---

### 🔲 Phase 2 — Mechanical & Enclosure Design
**Goal:** Define the physical form of the device before any electronics work.

**Tasks:**
- [ ] Sketch top-level form factor (hand drawing or CAD concept)
- [ ] Design food hopper shape and auger entry point
- [ ] Design water bowl / reservoir mount or integration
- [ ] Plan display and encoder panel face layout
- [ ] Plan camera position (v2 prep — reserve space now)
- [ ] Plan cable routing from PCB to all peripherals
- [ ] Decide: 3D printed enclosure vs. modified off-the-shelf box
- [ ] Draft enclosure 3D model (CAD) or detailed sketch with dimensions
- [ ] Plan mounting: countertop feet, anti-slip, wall bracket (if needed)

**Checkpoint 2 Exit Criteria:**
> Enclosure design is dimensionally defined. All components physically fit in the model/sketch. Auger mechanism concept is validated on paper.

---

### 🔲 Phase 3 — Schematic & Electrical Design
**Goal:** A complete electrical schematic before any wiring or PCB work begins.

**Tasks:**
- [ ] Draw full schematic (KiCad, EasyEDA, or equivalent)
- [ ] Map every GPIO pin — no conflicts, no floating pins
- [ ] Design power rails (5V and 3.3V; 12V if needed)
- [ ] Add decoupling capacitors on all power pins
- [ ] Add flyback diode on relay coil
- [ ] Add current-limiting resistors for LEDs and status indicators
- [ ] Add ESD / reverse-polarity protection on input connectors
- [ ] Define all connectors and cable types (JST, Dupont, screw terminal)
- [ ] Decide: perfboard prototype vs. custom PCB for v1
- [ ] If custom PCB: design PCB layout and order

**Checkpoint 3 Exit Criteria:**
> Schematic is complete, reviewed, and error-checked. Pin map is locked. Power budget is validated in schematic. Prototype build can begin.

---

### 🔲 Phase 4 — Prototype Hardware Assembly
**Goal:** A working, wired prototype of all subsystems on bench.

**Tasks:**
- [ ] Assemble power supply and verify 5V and 3.3V rails under load
- [ ] Wire and test ESP32-S3 (flash basic ESPHome, confirm Wi-Fi + HA discovery)
- [ ] Wire and test each sensor independently:
  - [ ] SHT31 (temp/humidity via I2C)
  - [ ] Food level sensor
  - [ ] Water level sensor
  - [ ] PIR / mmWave motion sensor
  - [ ] Rotary encoder and buttons
- [ ] Wire and test each output independently:
  - [ ] TFT display (render test image)
  - [ ] Stepper motor (rotate, reverse, hold)
  - [ ] Relay (switch on/off)
  - [ ] WS2812B LED strip (color test)
- [ ] Wire all subsystems together and confirm no conflicts
- [ ] First full ESPHome config — all entities visible in HA

**Checkpoint 4 Exit Criteria:**
> All hardware subsystems are functional and visible in Home Assistant. No pin conflicts. Prototype is stable on a bench without enclosure.

---

### 🔲 Phase 5 — Firmware Development
**Goal:** Full v1 feature firmware, running reliably on the prototype.

**Tasks:**
- [ ] Implement scheduled feeding (ESPHome time component + stepper trigger)
- [ ] Implement manual dispense (HA button entity → stepper)
- [ ] Implement portion sizing (configurable steps per dispense)
- [ ] Implement food low alert (threshold sensor → HA notification)
- [ ] Implement water low alert (binary sensor → HA notification)
- [ ] Implement display UI (clock, next feed time, food/water status icons)
- [ ] Implement local manual feed button (encoder or tactile button)
- [ ] Implement motion/presence reporting to HA
- [ ] Implement LED status logic (idle, feeding, alert, error states)
- [ ] Implement relay control from HA
- [ ] Implement jam/fault detection (stepper stall detection if possible)
- [ ] Implement power-loss recovery (state restored on boot)
- [ ] Implement OTA update flow and confirm it works
- [ ] Code review pass: clean up, comment, document all lambdas

**Checkpoint 5 Exit Criteria:**
> All v1 features work end-to-end. OTA updates work. Device recovers cleanly from power loss. No known critical bugs.

---

### 🔲 Phase 6 — Home Assistant Integration & Automations
**Goal:** HA is fully configured to use the device as intended.

**Tasks:**
- [ ] Create HA dashboard card for feeder (status, last feed, food/water levels)
- [ ] Create feeding schedule automations (time-based)
- [ ] Create low food / low water alert automations (mobile notification)
- [ ] Create motion-triggered automation (optional: camera snapshot placeholder)
- [ ] Create manual dispense script / button card
- [ ] Test all automations for edge cases (multiple triggers, overlapping schedules)
- [ ] Set up persistent notification or logbook entries for feed events

**Checkpoint 6 Exit Criteria:**
> HA dashboard is complete. All automations are tested and working. A non-technical user could operate the feeder from the HA app.

---

### 🔲 Phase 7 — Enclosure Build & Final Assembly
**Goal:** Move from bench prototype to finished device in its enclosure.

**Tasks:**
- [ ] 3D print or prepare final enclosure parts
- [ ] Install PCB / perfboard into enclosure
- [ ] Route and dress all cables (zip ties, cable channels)
- [ ] Mount display, encoder, and buttons to faceplate
- [ ] Install stepper motor and auger mechanism
- [ ] Install food hopper and water sensor
- [ ] Install LED strip
- [ ] Verify all components function after final assembly
- [ ] Run 24-hour soak test (continuous operation, scheduled feeds, HA monitoring)

**Checkpoint 7 Exit Criteria:**
> Device operates correctly in its final enclosure. 24-hour soak test passes with no faults. Physical build is clean and presentable.

---

### 🔲 Phase 8 — Testing, Validation & Documentation
**Goal:** The project is complete, documented, and ready for daily use.

**Tasks:**
- [ ] Write hardware documentation (schematic PDF, BOM, wiring notes)
- [ ] Write firmware documentation (ESPHome YAML comments, config guide)
- [ ] Write HA setup guide (how to add the device, automation templates)
- [ ] Document known limitations and workarounds
- [ ] Photograph and archive final build
- [ ] Create v2 backlog with prioritized features (see Section 7)
- [ ] Final git commit / backup of all source files

**Checkpoint 8 Exit Criteria:**
> Documentation is complete. All source files are backed up. Device is in daily production use. v2 backlog is defined.

---

## 6. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Stepper motor jams / skips steps (food clumping) | High | High | Add jam detection via step counter; test with actual kibble early |
| Food level sensor unreliable in dusty hopper | Medium | Medium | Test multiple sensor types in Phase 4; add manual override alert |
| ESP32-S3 GPIO count too tight | Low | Medium | Preliminary pin map shows sufficient GPIOs; use I2C expander if needed |
| ESPHome limitation blocks a feature | Low | Medium | Custom C++ lambda available as fallback |
| Display too slow for encoder UI | Low | Low | ILI9341 via SPI is fast; LVGL handles smooth rendering |
| Wi-Fi dropouts causing missed feeds | Low | High | ESPHome feeds are local/scheduled; offline feeding works without Wi-Fi |
| Power supply noise causing sensor errors | Medium | Medium | Decouple all rails in Phase 3 schematic |
| Enclosure doesn't fit all components | Medium | Medium | Phase 2 is fully mechanical — validate fit before ordering enclosure prints |

---

## 7. v2 Backlog

> Features explicitly deferred from v1. Revisit after Checkpoint 8.

| Priority | Feature | Notes |
|----------|---------|-------|
| High | **Live camera feed** | ESP32-S3 has camera interface; reserve GPIO in Phase 3 |
| High | **Power / energy monitoring** | Add INA219 or PZEM-004T on relay load circuit |
| Medium | **Food weight (load cell)** | HX711 + load cell under food bowl; more accurate than level sensor |
| Medium | **Automatic water top-up** | Second relay + peristaltic pump from reservoir |
| Low | **Pet recognition (camera ML)** | ESP-WHO framework or offload to HA AI |
| Low | **Companion mobile UI** | Native app or HA Companion widget |

---

## 8. Decisions Log

> Record all major decisions here as they are made during the project.

| Date | Decision | Rationale | Decided By |
|------|----------|-----------|------------|
| — | MCU: ESP32-S3 | GPIO count, ESPHome support, camera-ready for v2 | TBD in Phase 0 |
| — | Firmware: ESPHome | HA native integration, OTA, mature sensor support | TBD in Phase 0 |
| — | Display: ILI9341 TFT | Color, SPI speed, good library support | TBD in Phase 0 |
| — | Motion: PIR vs mmWave | PIR simpler; mmWave detects stationary pets | TBD in Phase 0 |
| — | Stepper driver: A4988 vs TMC2208 | TMC2208 is silent — important for sleeping pets nearby | TBD in Phase 0 |
| — | Power: single 5V vs dual 5V+12V | Depends on stepper motor selection | TBD in Phase 0 |

---

*Document version 1.0 — Created at project start. Update after each checkpoint.*
