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

Design and build a second-generation smart chicken coop monitor and door controller. The device will be reliable, locally controlled via Home Assistant, and expandable for future features including camera-based egg detection.

### Design Principles

- **Reliability first** — Door must never silently fail
- **Local-first** — Fully functional without cloud; HA is the hub
- **Maintainability** — Clean wiring, documented firmware, easy to reflash
- **Expandable** — Hardware and firmware designed with v2 features in mind (camera, weight sensing)
- **Solo-scale** — No over-engineering; keep scope tightly managed per phase

---

## 2. Feature Scope

### v1 — Must Have

| # | Feature | Notes |
|---|---------|-------|
| F1 | **Scheduled door** | Time-based open/close, configurable from HA |
| F2 | **Remote manual door** | Trigger from HA dashboard or mobile |
| F3 | **Food level alert** | Notify when hopper is low |
| F4 | **Water level alert** | Notify when bowl/reservoir is low |
| F5 | **Local display** | 2x 7-segment (TM1637 driver) showing daily egg count (0–99) |
| F6 | **Local input — egg count** | Rotary encoder: turn = ±1, push-hold 2s = reset to 0 |
| F7 | **Nest box occupancy** | LD2410 mmWave — detects stationary hen in nest box |
| F8 | **Temperature & humidity** | Environmental data reported to HA |
| F9 | **Status LEDs** | WS2812B strip for power, door state, alert states |
| F10 | **Door drive — horizontal leadscrew** | NEMA 17 48mm + TR8 leadscrew + TMC2208; door rides on heavy-duty drawer slide |
| F11 | **Relay output** | Switch a connected load (water pump, heat lamp, etc.) |
| F12 | **Wi-Fi OTA firmware updates** | Reflash without physical access |
| F13 | **Power loss recovery** | Door and egg count state restored correctly on reboot |

### v2 — Future (Scoped Out of v1)

- Camera feed + AI egg detection (automatic egg count logging)
- Power/energy monitoring of relay load
- Food weight measurement (load cell + HX711)
- Automatic water top-up (second relay + peristaltic pump)
- 4x4 keypad for direct numeric egg count entry
- Full TFT display (ILI9341) for richer local UI

---

## 3. Hardware Architecture Recommendation

### 3.1 Microcontroller — ESP32-S3

**Recommended MCU: Espressif ESP32-S3 (e.g., ESP32-S3-DevKitC-1)**

| Criteria | Rationale |
|----------|-----------|
| Wi-Fi | Built-in 2.4 GHz 802.11 b/g/n |
| GPIO count | 45 usable GPIOs — sufficient for all peripherals |
| I2C / SPI / UART | Hardware-native buses, multiple instances |
| Camera interface | DVP/MIPI CSI — GPIO reserved for v2 camera |
| ESPHome support | First-class — native HA integration |
| PSRAM | Up to 8 MB — handles future camera framebuffer |
| USB native | USB OTG for development and flashing |
| Power | 3.3V logic; pair with 5V → 3.3V LDO |

---

### 3.2 Door Mechanism — Horizontal Sliding (Locked)

**Chosen drive: TR8 leadscrew (8mm diameter, 8mm lead) + NEMA 17 48mm body + TMC2208**  
**Chosen guide: Heavy-duty full-extension ball-bearing drawer slide**

The coop door slides **horizontally**. The door panel parks beside the opening when open and slides across to cover it when closed.

#### Drawer Slide Selection

| Requirement | Specification |
|-------------|--------------|
| Type | Full-extension ball-bearing, **no soft-close** (detent interferes with limit switch and stepper) |
| Material | Stainless or zinc-plated for moisture resistance |
| Load rating | 100 lb+ (robust for outdoor use and a heavy door panel) |
| Travel length | Match to door opening width — confirm in Phase 2 (common: 12", 16", 18", 20", 24") |

> **Door panel sizing:** Panel width must be at least (opening width + slide overlap on fixed member) to fully close the opening. When open, the panel stacks to one side — confirm wall clearance in Phase 2.

#### Limit Switch Mounting

Both limit switches mount to the **fixed member** (wall/frame side) of the drawer slide. A small actuating tab or flag on the **moving member** (door side) trips the switches. This keeps all wiring stationary — no flex wiring to the moving door.

| Switch | Position | Function |
|--------|----------|----------|
| OPEN limit | Fixed member, at fully-retracted position | Homing reference — trips when door is fully open → zero step count |
| CLOSED limit | Fixed member, at fully-extended position | Safety cutoff — trips when door reaches full-closed travel |

#### Motor and Leadscrew

| Component | Spec | Notes |
|-----------|------|-------|
| Stepper | NEMA 17 48mm body (e.g. 17HS8401), 1.8A, ~65 N·cm | Higher torque than 40mm variant; well within TMC2208 2A limit |
| Driver | TMC2208 | Silent operation at 12V; important near animals |
| Leadscrew | TR8, 8mm lead | Self-locking against wind load; holds position without power |
| Coupler | Flexible jaw coupler (5mm to 8mm) | Absorbs minor motor/leadscrew misalignment |
| Mounting | Motor fixed to coop frame at one end; far-end bearing block supports free end of leadscrew | TR8 nut bracket fixed to door panel or slide moving member |

> **Horizontal orientation note:** Gravity is no longer a failure mode. The TR8 self-locking property still provides value — it holds the door position against wind without the motor holding torque, and resists any unintended movement.

---

### 3.3 Peripheral Component Selections

#### Sensors & Inputs

| Component | Part | Bus / Interface | Notes |
|-----------|------|-----------------|-------|
| Temp / Humidity | SHT31 or AHT21 | I2C | Accurate and stable in outdoor/coop environment |
| Food level | HC-SR04 ultrasonic (on hand) | GPIO trigger/echo | Distance from sensor to food surface; 5-sample median filter |
| Water level | XKC-Y25 capacitive sensor | GPIO (digital) | No exposed electrodes — safe around water |
| Nest box occupancy | LD2410 mmWave | UART | Detects stationary hen; superior to PIR for this use |
| Door limit (open) | Roller microswitch SS-5GL | GPIO (digital in, pull-up) | Home reference; trips when door fully open → zero step count |
| Door limit (closed) | Roller microswitch SS-5GL | GPIO (digital in, pull-up) | Safety cutoff at full-closed travel |
| Rotary encoder | EC11 with push button | GPIO (A, B, SW) | Egg count: CW/CCW = ±1, push-hold 2s = reset |

#### Outputs & Actuators

| Component | Part | Interface | Notes |
|-----------|------|-----------|-------|
| 7-segment display | TM1637 module (2-digit) | CLK + DIO (2 GPIOs) | Egg count 0–99; ESPHome native component |
| Door stepper | NEMA 17 48mm, 17HS8401 or equivalent | STEP / DIR / EN | Door drive via TR8 leadscrew |
| Stepper driver | TMC2208 | STEP / DIR / EN (GPIO) | Silent stepping at 12V |
| Relay | 5V single-channel relay module | GPIO | Controls pump, lamp, or other load |
| LED strip | WS2812B addressable | GPIO (RMT 1-wire) | Door state, alerts, ambient status |
| Buzzer (optional) | Passive piezo | GPIO (PWM) | Audio alerts; may be omitted |

#### Power

| Rail | Source | Consumers |
|------|--------|-----------|
| 5V | 5V 3A wall adapter, or 12V main + LM2596 buck converter | Relay coil, TM1637, logic |
| 3.3V | AMS1117-3.3 LDO from 5V | ESP32-S3, sensors, encoder |
| 12V (motor) | 12V 2A supply, or 12V main with shared 5V buck | NEMA 17 via TMC2208 |

> **Power architecture open:** Decide in Phase 1 — single 12V main + LM2596 5V buck is the cleaner single-cable solution.

---

### 3.4 Bus / GPIO Map (Preliminary)

| Bus | Devices |
|-----|---------|
| I2C (Bus 0) | SHT31 temp/humidity |
| UART 0 | LD2410 mmWave sensor |
| UART 1 | Reserved for v2 camera or debug |
| CLK + DIO | TM1637 7-segment (2 GPIOs) |
| GPIO digital in | HC-SR04 trigger/echo, XKC-Y25, door limits ×2, encoder A/B/SW |
| GPIO digital out | Relay, stepper STEP/DIR/EN, buzzer |
| GPIO RMT | WS2812B LED strip |
| Camera DVP header | Reserved, unpopulated in v1 |

> Final GPIO pin assignments are a Phase 3 deliverable.

---

## 4. Firmware & Integration Stack

### 4.1 Firmware Platform — ESPHome

| Criteria | Rationale |
|----------|-----------|
| HA integration | Native ESPHome API — auto-discovery, no MQTT needed |
| OTA updates | Built-in, triggered from HA or web UI |
| Stepper | `stepper:` component — native A4988/TMC2208 support |
| Sensors | First-class: SHT31, HC-SR04, I2C, UART, binary sensors |
| TM1637 | `display: platform: tm1637` |
| Rotary encoder | `rotary_encoder:` component — built in |
| NVS persistence | `globals: restore_value: true` for egg count and last door state |
| Custom logic | Lambda C++ for door state machine, homing routine, time-aware boot |

### 4.2 Home Assistant Integration Points

| HA Entity | Type | Source |
|-----------|------|--------|
| `sensor.coop_temperature` | Sensor | SHT31 |
| `sensor.coop_humidity` | Sensor | SHT31 |
| `sensor.coop_food_distance_cm` | Sensor | HC-SR04 |
| `binary_sensor.coop_food_low` | Binary Sensor | HC-SR04 threshold |
| `binary_sensor.coop_water_low` | Binary Sensor | XKC-Y25 |
| `binary_sensor.coop_nest_occupied` | Binary Sensor | LD2410 mmWave |
| `number.coop_egg_count` | Number | Rotary encoder, synced to HA |
| `button.coop_door_open` | Button | Triggers door open sequence |
| `button.coop_door_close` | Button | Triggers manual door close (schedule resumes) |
| `sensor.coop_door_state` | Text Sensor | open / closed / moving / error |
| `switch.coop_relay` | Switch | Controls relay output |
| `light.coop_leds` | Light | WS2812B strip |

### 4.3 Door State Machine

**Default state: OPEN.** The door opens to safe (chickens can free-range) and closes only on schedule or explicit manual command.

```
Power-on / reboot:
  └── [HOMING] → drive toward OPEN limit switch (slow speed)
        ├── OPEN limit tripped → zero step count → [OPEN]  (daytime or unknown time)
        └── (open decision) See boot behavior note below

[OPEN] ──── schedule close / manual close ────→ [CLOSING] ──→ [CLOSED]
  ↑                                                                  │
  └──────── schedule open / manual open ──── [OPENING] ←────────────┘

Manual close behavior:
  - Closes door immediately
  - Does NOT cancel or modify the open schedule
  - Next scheduled open fires normally

Fault conditions (any state):
  ├── Either limit switch tripped unexpectedly during travel → [ERROR] + HA alert
  ├── Step count reaches expected position but limit not confirmed → [ERROR] + HA alert
  └── [ERROR] state requires manual HA clear; door does not auto-recover
```

> **⚠ Open decision on power-loss reboot — decision required:**  
> If power is lost while the door is closed (e.g. overnight), the firmware will home to OPEN on reboot.  
> **Option A:** Always home to OPEN on boot (simple; may open door at an unsafe hour after a brief power blip).  
> **Option B:** Check time on boot — if within the scheduled closed window, skip homing and restore CLOSED state; home to OPEN when the scheduled open time arrives.  
> **Option C:** Restore last known door state from NVS, skip homing unless state is unknown.  
> *This decision must be made before Phase 5 firmware work begins.*

### 4.4 Egg Count Logic

```
Rotary encoder CW        → egg_count + 1 (max 99)
Rotary encoder CCW       → egg_count - 1 (min 0)
Push-hold 2 seconds      → egg_count reset to 0 (debounced, prevents accidental wipe)
On any change            → update TM1637 display + sync number.coop_egg_count to HA
Midnight (HA automation) → number.set_value resets count to 0
Power-on restore         → globals restore_value reads last count from NVS flash
```

---

## 5. Phase Breakdown & Checkpoints

### ✅ Phase 0 — Project Foundation

**Locked:**
- [x] MCU: ESP32-S3, Firmware: ESPHome
- [x] Display: TM1637 7-segment (no TFT in v1)
- [x] Local input: rotary encoder only (keypad deferred to v2)
- [x] Food level sensor: HC-SR04 (on hand)
- [x] Door orientation: horizontal sliding
- [x] Door guide: heavy-duty full-extension drawer slide (no soft-close)
- [x] Door drive: TR8 leadscrew + NEMA 17 48mm (17HS8401) + TMC2208
- [x] Homing: jog to OPEN limit switch; zero at open position
- [x] Default door state: OPEN; close only on schedule or manual command
- [x] Manual close: does not cancel open schedule
- [x] Occupancy sensor: LD2410 mmWave
- [x] Enclosure: Hoffman A806CH (6"×8"); A1008CH fallback
- [x] Hopper measurement: HC-SR04 distance from top
- [x] Camera GPIO/header reserved for v2 (unpopulated in v1)

**Open:**
- [ ] Power architecture: 12V main + 5V buck vs dual supply — **decide Phase 1**
- [ ] Door boot behavior after power loss while closed (Options A/B/C above) — **decide Phase 1**
- [ ] Drawer slide travel length — depends on door opening width — **confirm Phase 2**

**Checkpoint 0 Exit Criteria:**
> All hardware and feature decisions documented. Power architecture and boot behavior decisions made. No ambiguity on v1 scope.

---

### 🔲 Phase 1 — Component Selection & BOM

**Tasks:**
- [ ] Decide and document power architecture
- [ ] Decide door boot behavior on power-loss reboot
- [ ] Source TR8 leadscrew, NEMA 17 17HS8401, TMC2208, flexible coupler, bearing block
- [ ] Select drawer slide (full-extension, no soft-close, 100 lb+, appropriate travel length)
- [ ] Select TM1637 module; verify faceplate fit
- [ ] Select LD2410 mmWave module
- [ ] Create full BOM: part, qty, supplier, unit cost, link
- [ ] Calculate power budget (mA per rail, worst case with motor stall current)
- [ ] Identify long-lead-time parts; order early
- [ ] Order all components

**Checkpoint 1 Exit Criteria:**
> BOM complete. All parts ordered. Power budget validates supply choice. Boot behavior decision documented.

---

### 🔲 Phase 2 — Mechanical & Enclosure Design

**Tasks:**
- [ ] Measure door opening width; confirm drawer slide travel length
- [ ] Determine door panel dimensions (width ≥ opening + slide overlap; height to match opening)
- [ ] Determine door panel material (aluminum sheet, plywood, PVC board)
- [ ] Plan motor mount at one end of leadscrew; far-end bearing block at other end
- [ ] Plan TR8 nut bracket on door panel / slide moving member
- [ ] Plan limit switch positions on fixed member; actuating tab on moving member
- [ ] Confirm wall clearance for parked (open) door panel
- [ ] Plan HC-SR04 mount inside hopper; confirm beam angle and range
- [ ] Plan water sensor integration
- [ ] Plan TM1637 display and encoder faceplate layout on enclosure
- [ ] Reserve camera mount space (v2 prep)
- [ ] Plan cable routing from enclosure to door assembly, hopper, nest box
- [ ] Confirm all electronics fit in Hoffman A806CH

**Checkpoint 2 Exit Criteria:**
> Door mechanism dimensionally defined. All components fit. Motor mount, leadscrew, and slide layout validated on paper or CAD.

---

### 🔲 Phase 3 — Schematic & Electrical Design

**Tasks:**
- [ ] Draw full schematic (KiCad or EasyEDA)
- [ ] Map every GPIO — no conflicts, no floating inputs
- [ ] Add pull-ups on all digital inputs (limit switches, encoder, water sensor)
- [ ] Design power rails (5V, 3.3V, 12V)
- [ ] Add decoupling capacitors on all power pins
- [ ] Add flyback diode on relay coil
- [ ] Add current-limiting resistors for status LEDs
- [ ] Add RC debounce on limit switch inputs
- [ ] Add ESD / reverse-polarity protection on external connectors
- [ ] Reserve unpopulated camera header (DVP pinout)
- [ ] Define all connectors (JST for sensors, screw terminals for motor and power)
- [ ] Decide: perfboard prototype vs. custom PCB

**Checkpoint 3 Exit Criteria:**
> Schematic complete and error-checked. GPIO map locked. Power budget validated. Prototype build can begin.

---

### 🔲 Phase 4 — Prototype Hardware Assembly

**Tasks:**
- [ ] Assemble power supply; verify all rails under load
- [ ] Flash basic ESPHome; confirm Wi-Fi + HA discovery
- [ ] Test each sensor:
  - [ ] SHT31 (I2C, temp + humidity)
  - [ ] HC-SR04 (distance, median filter)
  - [ ] XKC-Y25 (water level)
  - [ ] LD2410 mmWave (occupancy, UART config)
  - [ ] Rotary encoder (A/B/SW)
  - [ ] Both limit switches (GPIO logic levels and pull-up behavior)
- [ ] Test each output:
  - [ ] TM1637 (digit render 0–9)
  - [ ] NEMA 17 + TMC2208 (rotate, reverse, homing to OPEN limit switch)
  - [ ] Relay (on/off)
  - [ ] WS2812B (color test)
- [ ] Wire all subsystems together; confirm no conflicts
- [ ] Full ESPHome config — all entities in HA

**Checkpoint 4 Exit Criteria:**
> All subsystems functional in HA. Homing to OPEN limit works on bench. No pin conflicts.

---

### 🔲 Phase 5 — Firmware Development

**Tasks:**
- [ ] Door state machine (Section 4.3)
  - [ ] Homing routine: slow drive to OPEN limit → zero step count → [OPEN]
  - [ ] Implement boot behavior chosen in Phase 1 (time-aware or NVS-restore)
  - [ ] Open/close sequences with step counting
  - [ ] Limit switch interrupt handlers (unexpected trip → ERROR + HA alert)
  - [ ] Door state entity synced to HA
- [ ] Scheduled door (time component + state machine trigger)
- [ ] Manual door open and close (HA buttons → state machine; close does not cancel schedule)
- [ ] Food low alert (HC-SR04 threshold → HA)
- [ ] Water low alert (XKC-Y25 → HA)
- [ ] Egg count:
  - [ ] CW/CCW increment/decrement (0–99)
  - [ ] Hold-push 2s reset with debounce
  - [ ] NVS persistence
  - [ ] TM1637 display update
  - [ ] HA number entity sync
- [ ] Nest box occupancy (LD2410 → HA binary sensor)
- [ ] LED status logic (idle/open, closing, opening, alert, error)
- [ ] Relay control from HA
- [ ] OTA update flow confirmed
- [ ] Code review and lambda documentation

**Checkpoint 5 Exit Criteria:**
> All v1 features work end-to-end. Door state machine reliable. Homing and boot behavior tested. Power-loss recovery tested. No critical bugs.

---

### 🔲 Phase 6 — Home Assistant Integration & Automations

**Tasks:**
- [ ] HA dashboard (door state, food/water, egg count, temp/humidity, nest occupancy)
- [ ] Door schedule automations (open and close)
- [ ] Food/water alert automations (mobile push notification)
- [ ] Midnight egg count reset automation
- [ ] Nest occupancy log automation (placeholder for v2 camera trigger)
- [ ] Test all automations for edge cases (power-loss recovery, schedule during manual close)
- [ ] Logbook entries for door events and daily egg totals

**Checkpoint 6 Exit Criteria:**
> Dashboard and all automations complete and tested, including edge cases.

---

### 🔲 Phase 7 — Enclosure Build & Final Assembly

**Tasks:**
- [ ] Install PCB into Hoffman A806CH enclosure
- [ ] Mount NEMA 17 at end of leadscrew; install bearing block at far end
- [ ] Install drawer slide; attach TR8 nut bracket to moving member / door panel
- [ ] Mount limit switches on fixed member; attach actuating tabs to moving member
- [ ] Route and dress all cables (no flex wiring to moving door — all wiring to fixed member)
- [ ] Mount TM1637 and encoder on faceplate
- [ ] Install HC-SR04 in hopper, water sensor, LED strip
- [ ] Verify all functions after final assembly
- [ ] Run 24-hour soak test (door cycles, sensors, HA monitoring)

**Checkpoint 7 Exit Criteria:**
> Device operates in enclosure. 24-hour soak test passes. Cables are clean and serviceable.

---

### 🔲 Phase 8 — Testing, Validation & Documentation

**Tasks:**
- [ ] Hardware documentation (schematic PDF, BOM, wiring, door mechanism photos)
- [ ] Firmware documentation (YAML comments, config guide, state machine diagram)
- [ ] HA setup guide (device add, automation templates)
- [ ] Document known limitations and workarounds
- [ ] Photograph final build and door mechanism
- [ ] Final git commit / backup all source files
- [ ] Define and prioritize v2 backlog

**Checkpoint 8 Exit Criteria:**
> Documentation complete. Files backed up. Device in daily use. v2 backlog defined.

---

## 6. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Drawer slide binds or corrodes outdoors | Medium | High | Use stainless or zinc-plated slide; inspect and lubricate during 24-hr soak and periodically |
| Soft-close detent on slide fights stepper | High (if wrong slide chosen) | Medium | Explicitly select no-soft-close slide in Phase 1 BOM |
| Door panel misaligned on slide (racking) | Medium | Medium | Ensure TR8 nut bracket and slide moving member are both attached to same rigid panel; validate in Phase 2 |
| HC-SR04 unreliable in dusty hopper | Medium | Medium | 5-sample median filter; test in real conditions early |
| LD2410 false positives in nest box | Medium | Low | Tune sensitivity zone; false positives don't trigger actuators |
| Reboot during closed window opens door | Medium | High | Resolve with Phase 1 boot behavior decision; Option B or C strongly recommended |
| NEMA 17 draws too much current on stall | Low | Low | TMC2208 stall guard can detect stall; current limit set via Vref resistor |
| ESP32-S3 GPIO count too tight | Low | Medium | Preliminary map shows sufficient; I2C expander as fallback |
| Wi-Fi dropout causes missed door schedule | Low | High | ESPHome time component caches schedule locally; offline operation works |
| Hoffman A806CH too tight for wiring | Medium | Medium | Validate in Phase 2; A1008CH (10"×8") is fallback |
| Motor rail noise affecting sensor readings | Medium | Medium | Decouple all rails; separate motor and sensor traces in Phase 3 |

---

## 7. v2 Backlog

| Priority | Feature | Notes |
|----------|---------|-------|
| High | **Camera + AI egg detection** | ESP32-S3 camera interface reserved; auto egg count logging |
| High | **Power / energy monitoring** | INA219 or PZEM-004T on relay load |
| Medium | **Food weight (load cell)** | HX711 + load cell; more precise than distance sensor |
| Medium | **Automatic water top-up** | Second relay + peristaltic pump |
| Medium | **TFT display (ILI9341)** | Richer local UI if 7-segment proves limiting |
| Low | **4x4 keypad** | Direct numeric entry; only if encoder proves awkward |
| Low | **Pet/predator recognition** | Camera AI via ESP-WHO or HA offload |

---

## 8. Decisions Log

| Date | Decision | Rationale | Status |
|------|----------|-----------|--------|
| — | MCU: ESP32-S3 | GPIO count, ESPHome support, camera-ready for v2 | Locked |
| — | Firmware: ESPHome | HA native integration, OTA, mature sensor support | Locked |
| — | Door orientation: horizontal sliding | Space constraints; eliminates vertical freefall concern | Locked |
| — | Door guide: heavy-duty drawer slide (no soft-close) | Off-the-shelf, robust, no custom fabrication; no soft-close to avoid detent interference | Locked |
| — | Door drive: TR8 leadscrew + NEMA 17 48mm + TMC2208 | TR8 self-locking holds against wind; 17HS8401 gives 65 N·cm torque headroom | Locked |
| — | Homing: jog to OPEN limit, zero at open | Safe default — chickens can free-range; closed only on schedule or manual command | Locked |
| — | Manual close: does not cancel open schedule | Schedule resumes after manual close; consistent behavior | Locked |
| — | Display: TM1637 7-segment (2-digit) | 2 GPIOs vs 10–16 direct drive; ESPHome native | Locked |
| — | Local input: rotary encoder only | 3 GPIOs, covers all v1 local input; keypad to v2 | Locked |
| — | Occupancy sensor: LD2410 mmWave | Detects stationary hen; PIR misses still animals | Locked |
| — | Food level: HC-SR04 | On hand; contactless; apply median filter | Locked |
| — | Enclosure: Hoffman A806CH (6"×8") | Must fit; A1008CH fallback | Locked |
| — | Camera: GPIO reserved, header unpopulated in v1 | v2 egg detection path; zero firmware cost in v1 | Locked |
| — | Power architecture | 12V+buck vs dual-supply | **Open — Phase 1** |
| — | Boot behavior after power-loss while closed | Options A (always open), B (time-aware), C (NVS restore) — see Section 4.3 | **Open — Phase 1** |

---

*Document version 1.3 — Horizontal sliding door locked. Drawer slide selected. Homing revised to OPEN position. NEMA 17 48mm (17HS8401) locked. Manual close / schedule behavior defined. Boot behavior after power-loss flagged as open decision.*