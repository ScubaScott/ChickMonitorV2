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

- **Reliability first** — Door fails open; chickens are never trapped by a fault
- **Local-first** — Fully functional without cloud; HA is the hub
- **Maintainability** — Clean wiring, documented firmware, easy to reflash
- **Expandable** — Hardware and firmware designed with v2 features in mind
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
| F13 | **Power loss recovery** | Fail-open by design; time-aware post-homing schedule check |

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

### 3.2 Door Mechanism (Locked)

**Drive:** TR8 leadscrew (8mm diameter, 8mm lead) + NEMA 17 48mm (17HS8401, ~65 N·cm) + TMC2208  
**Guide:** Heavy-duty full-extension ball-bearing drawer slide — 16" travel recommended  
**Orientation:** Horizontal sliding; door parks to motor side when open

#### Door Geometry

| Dimension | Value | Notes |
|-----------|-------|-------|
| Opening width | 12" | Confirmed |
| Opening height | 12" | Confirmed |
| Far-side wall lip | 2" | Door overlaps this when closed |
| Motor-side clearance | Several feet | Door parks here when open |
| Recommended slide travel | 16" | 12" clear travel + 4" overlap margin on motor side |
| Door panel width | 13–14" | ½–1" frame overlap each side when closed |
| Door panel height | ~13" | Slight overlap top and bottom |
| Leadscrew travel required | ~305mm (12") | 8mm/rev × ~38 rev = ~7,600 steps at 1/16 microstepping |

#### Drawer Slide Selection

| Requirement | Specification |
|-------------|--------------|
| Type | Full-extension ball-bearing, **no soft-close** (detent interferes with stepper and limit switch) |
| Travel | 16" recommended (minimum 14") |
| Material | Stainless or zinc-plated for moisture resistance |
| Load rating | 100 lb+ |

#### Limit Switch Mounting

Both switches mount to the **fixed member** (wall/frame side). A passive actuating tab on the **moving member** (door side) trips each switch. All wiring is stationary — no flex wiring to the moving door.

| Switch | Position | Function |
|--------|----------|----------|
| OPEN limit | Fixed member, fully-retracted position | Homing reference: trips when door fully open → zero step count |
| CLOSED limit | Fixed member, fully-extended position | Safety cutoff at full-closed travel; also confirms closed state |

#### Motor and Leadscrew

| Component | Spec | Notes |
|-----------|------|-------|
| Stepper | NEMA 17 48mm, 17HS8401, ~1.8A, ~65 N·cm | Well within TMC2208 2A limit; significant torque headroom for 12" horizontal load |
| Driver | TMC2208 at 12V | Silent operation; Vref set for ~1.2A RMS (safe operating margin) |
| Leadscrew | TR8, 8mm lead | Self-locking holds door against wind without motor power |
| Coupler | Flexible jaw coupler (5mm to 8mm) | Absorbs motor/leadscrew misalignment |
| Motor mount | Fixed to coop frame, motor side | Aligned parallel to slide |
| Far-end support | Bearing block (SK8 or equivalent) | Supports free end of leadscrew; prevents whip |
| TR8 nut bracket | Fixed to door panel or slide moving member | Door translates as leadscrew turns |

---

### 3.3 Peripheral Component Selections

#### Sensors & Inputs

| Component | Part | Bus / Interface | Notes |
|-----------|------|-----------------|-------|
| Temp / Humidity | SHT31 or AHT21 | I2C | Stable in outdoor/coop environment |
| Food level | HC-SR04 (on hand) | GPIO trigger/echo | Distance from sensor to food surface; 5-sample median filter |
| Water level | XKC-Y25 capacitive | GPIO (digital) | No exposed electrodes |
| Nest box occupancy | LD2410 mmWave | UART | Detects stationary hen |
| Door limit (open) | Roller microswitch SS-5GL | GPIO (digital in, pull-up) | Homing reference; zero at full open |
| Door limit (closed) | Roller microswitch SS-5GL | GPIO (digital in, pull-up) | Safety cutoff + closed confirmation |
| Rotary encoder | EC11 with push button | GPIO (A, B, SW) | Egg count ±1; push-hold 2s = reset |

#### Outputs & Actuators

| Component | Part | Interface | Notes |
|-----------|------|-----------|-------|
| 7-segment display | TM1637 module (2-digit) | CLK + DIO (2 GPIOs) | Egg count 0–99 |
| Door stepper | NEMA 17 48mm, 17HS8401 | STEP / DIR / EN | Horizontal door via TR8 |
| Stepper driver | TMC2208 | STEP / DIR / EN | Silent; 12V supply |
| Relay | 5V single-channel relay module | GPIO | Pump, lamp, or other load |
| LED strip | WS2812B addressable | GPIO (RMT) | Door state and alert status |
| Buzzer (optional) | Passive piezo | GPIO (PWM) | May be omitted |

#### Power

| Rail | Source | Consumers |
|------|--------|-----------|
| 12V (main) | 12V 2A wall adapter | NEMA 17 via TMC2208 |
| 5V | LM2596 buck converter from 12V main | Relay coil, TM1637, logic |
| 3.3V | AMS1117-3.3 LDO from 5V | ESP32-S3, sensors, encoder |

> **Power architecture locked:** Single 12V main supply with LM2596 5V buck. One wall cable to enclosure; all rails derived internally. Simplifies enclosure entry and wiring.

---

### 3.4 Bus / GPIO Map (Preliminary)

| Bus | Devices |
|-----|---------|
| I2C (Bus 0) | SHT31 temp/humidity |
| UART 0 | LD2410 mmWave |
| UART 1 | Reserved for v2 camera / debug |
| CLK + DIO | TM1637 7-segment (2 GPIOs) |
| GPIO digital in | HC-SR04 trigger/echo, XKC-Y25, door limits ×2, encoder A/B/SW |
| GPIO digital out | Relay, stepper STEP/DIR/EN, buzzer |
| GPIO RMT | WS2812B LED strip |
| Camera DVP header | Reserved, unpopulated in v1 |

> Final GPIO pin assignments are a Phase 3 deliverable.

---

## 4. Firmware & Integration Stack

### 4.1 Firmware Platform — ESPHome

| Feature | Implementation |
|---------|---------------|
| HA integration | Native ESPHome API — auto-discovery, no MQTT needed |
| OTA updates | Built-in |
| Stepper | `stepper:` component — native TMC2208/A4988 support |
| Sensors | SHT31, HC-SR04, I2C, UART, binary sensors — first-class |
| TM1637 | `display: platform: tm1637` |
| Rotary encoder | `rotary_encoder:` component |
| NVS persistence | `globals: restore_value: true` — egg count only; **no door state written to NVS** |
| Time | `time: platform: sntp` with validity check in lambda |
| Custom logic | Lambda C++ for door state machine, homing routine, boot sequence |

### 4.2 Home Assistant Integration Points

| HA Entity | Type | Source |
|-----------|------|--------|
| `sensor.coop_temperature` | Sensor | SHT31 |
| `sensor.coop_humidity` | Sensor | SHT31 |
| `sensor.coop_food_distance_cm` | Sensor | HC-SR04 |
| `binary_sensor.coop_food_low` | Binary Sensor | HC-SR04 threshold |
| `binary_sensor.coop_water_low` | Binary Sensor | XKC-Y25 |
| `binary_sensor.coop_nest_occupied` | Binary Sensor | LD2410 |
| `number.coop_egg_count` | Number | Rotary encoder, synced to HA |
| `button.coop_door_open` | Button | Manual open command |
| `button.coop_door_close` | Button | Manual close (schedule resumes unchanged) |
| `sensor.coop_door_state` | Text Sensor | open / closed / moving / homing / error |
| `switch.coop_relay` | Switch | Controls relay output |
| `light.coop_leds` | Light | WS2812B strip |

### 4.3 Door State Machine

**Safety contract: the door always fails open. No door state is written to NVS.**

#### Boot / Power-Loss Recovery Sequence

```
[BOOT]
  └──→ [HOMING] — drive toward OPEN limit at reduced speed
         └── OPEN limit tripped → zero step count → door is physically OPEN
               └──→ [SNTP CHECK] — wait for time sync (max 30s timeout)
                     ├── Time confirmed valid
                     │     └── Is current time within a scheduled-close window?
                     │           ├── YES → [CLOSING] → [CLOSED]  (schedule resumes normally)
                     │           └── NO  → [OPEN]  (await schedule or manual command)
                     └── Time not confirmed after timeout
                           └── [OPEN]  (fail-safe; schedule will close when time syncs)
```

> The door is physically open for the duration of the homing move (~5–10 seconds) plus the SNTP check window on every reboot. This is the intentional fail-safe behavior. A brief door open during a nighttime power blip is acceptable; trapping chickens is not.

#### Normal Operating State Machine

```
[OPEN] ──── scheduled close / manual close ────→ [CLOSING] ──→ [CLOSED]
  ↑                                                                    │
  └──────── scheduled open / manual open ──── [OPENING] ←────────────┘

Manual close: closes door immediately; does NOT modify or cancel the open schedule.
Manual open:  opens door immediately; does NOT modify or cancel the close schedule.
```

#### Error / Fault Handling

```
Any state:
  ├── Limit switch tripped unexpectedly during travel
  │     └──→ [ERROR] — motor cut immediately → HA alert sent
  ├── Motor reaches expected step count but expected limit not confirmed
  │     └──→ [ERROR] — HA alert sent
  └── [ERROR] state:
        ├── Door physically remains wherever it stopped (open or closed)
        ├── HA notification with error description
        ├── Requires manual clear via HA button before resuming automation
        └── Manual open/close commands still accepted in ERROR state
```

### 4.4 Egg Count Logic

```
Rotary encoder CW        → egg_count + 1 (max 99)
Rotary encoder CCW       → egg_count - 1 (min 0)
Push-hold 2 seconds      → egg_count reset to 0 (debounced)
On any change            → update TM1637 + sync number.coop_egg_count to HA
Midnight (HA automation) → number.set_value resets to 0
Power-on restore         → globals restore_value from NVS (egg count only)
```

---

## 5. Phase Breakdown & Checkpoints

### ✅ Phase 0 — Project Foundation

**Locked:**
- [x] MCU: ESP32-S3, Firmware: ESPHome
- [x] Display: TM1637 7-segment, 2-digit (no TFT in v1)
- [x] Local input: rotary encoder only
- [x] Food level: HC-SR04 (on hand)
- [x] Door orientation: horizontal sliding
- [x] Door guide: heavy-duty full-extension drawer slide, 16" travel, no soft-close
- [x] Door opening: 12" × 12"; far-side lip 2"; motor-side clearance several feet
- [x] Door panel: 13–14" wide; leadscrew travel ~305mm (~7,600 steps at 1/16)
- [x] Door drive: TR8 leadscrew + NEMA 17 48mm (17HS8401) + TMC2208
- [x] Homing: always to OPEN limit on boot; zero at open
- [x] Boot behavior: home to OPEN → SNTP check (30s timeout) → close if in schedule window; else stay open
- [x] No door state written to NVS
- [x] Manual close: does not cancel open schedule
- [x] Occupancy: LD2410 mmWave
- [x] Power: 12V main + LM2596 5V buck; 3.3V LDO from 5V
- [x] Enclosure: Hoffman A806CH (6"×8"); A1008CH fallback
- [x] Camera GPIO/header reserved for v2

**Open:**
- [ ] Drawer slide specific part number and source — **Phase 1**
- [ ] Door panel material — **Phase 2**

**Checkpoint 0 Exit Criteria:**
> All hardware, firmware, and behavior decisions documented and locked. No ambiguity on v1 scope.

---

### 🔲 Phase 1 — Component Selection & BOM

**Tasks:**
- [ ] Source TR8 leadscrew (≥350mm), NEMA 17 17HS8401, TMC2208, flexible coupler, SK8 bearing block
- [ ] Select and order 16" full-extension no-soft-close slide (stainless or zinc, 100 lb+)
- [ ] Select limit switches (SS-5GL or equivalent); design actuating tab
- [ ] Select TM1637 module; verify faceplate fit in A806CH
- [ ] Select LD2410 mmWave module
- [ ] Select LM2596 5V buck module
- [ ] Create full BOM: part, qty, supplier, unit cost, link
- [ ] Calculate power budget (mA per rail, worst case with 1.2A RMS motor)
- [ ] Confirm 12V 2A supply is adequate (motor stall + all logic)
- [ ] Identify long-lead parts; order early
- [ ] Order all components

**Checkpoint 1 Exit Criteria:**
> BOM complete. All parts ordered. Power budget confirms 12V 2A is adequate.

---

### 🔲 Phase 2 — Mechanical & Enclosure Design

**Tasks:**
- [ ] Determine door panel material (aluminum sheet, PVC board, or exterior plywood)
- [ ] Fabricate or source door panel (13–14" wide × ~13" tall)
- [ ] Plan motor mount at one end of coop frame; leadscrew parallel to slide
- [ ] Plan SK8 bearing block position at far end of leadscrew
- [ ] Attach TR8 nut bracket to door panel / slide moving member
- [ ] Plan limit switch positions on fixed member; fabricate actuating tabs on moving member
- [ ] Verify door clears opening fully when open; verify 2" lip overlap when closed
- [ ] Plan HC-SR04 mount angle inside hopper
- [ ] Plan water sensor mount
- [ ] Plan TM1637 and encoder position on A806CH faceplate
- [ ] Reserve camera position (v2)
- [ ] Plan cable routing: all door wiring to fixed frame; no flex wiring to door panel

**Checkpoint 2 Exit Criteria:**
> Door mechanism dimensionally validated. All components fit. No cable routing to moving parts.

---

### 🔲 Phase 3 — Schematic & Electrical Design

**Tasks:**
- [ ] Draw full schematic (KiCad or EasyEDA)
- [ ] Map every GPIO — no conflicts, no floating inputs
- [ ] Pull-ups on all digital inputs (limit switches, encoder SW, water sensor)
- [ ] RC debounce on limit switch inputs
- [ ] Design power rails: 12V in → LM2596 5V → AMS1117 3.3V
- [ ] Decoupling capacitors on all power pins
- [ ] Flyback diode on relay coil
- [ ] Current-limiting resistors for status LEDs
- [ ] ESD / reverse-polarity protection on external connectors
- [ ] Reserve camera header (DVP pinout, unpopulated)
- [ ] Define connectors: JST-PH for sensors, screw terminals for motor/power
- [ ] Decide: perfboard vs. custom PCB

**Checkpoint 3 Exit Criteria:**
> Schematic complete and reviewed. GPIO map locked. Power budget validated in schematic.

---

### 🔲 Phase 4 — Prototype Hardware Assembly

**Tasks:**
- [ ] Assemble and verify power rails under load
- [ ] Flash basic ESPHome; confirm Wi-Fi + HA discovery
- [ ] Test each sensor individually:
  - [ ] SHT31 I2C readings
  - [ ] HC-SR04 distance with median filter
  - [ ] XKC-Y25 water level
  - [ ] LD2410 occupancy detection via UART
  - [ ] Rotary encoder A/B/SW
  - [ ] Both limit switches (logic levels, pull-up behavior)
- [ ] Test each output individually:
  - [ ] TM1637 digit render 0–9
  - [ ] NEMA 17 + TMC2208: rotate, reverse, homing to OPEN limit
  - [ ] Relay on/off
  - [ ] WS2812B color test
- [ ] Wire all subsystems; confirm no conflicts
- [ ] Full ESPHome config — all entities visible in HA

**Checkpoint 4 Exit Criteria:**
> All subsystems functional in HA. Homing to OPEN limit works on bench. No pin conflicts.

---

### 🔲 Phase 5 — Firmware Development

**Tasks:**
- [ ] Boot / homing sequence (Section 4.3):
  - [ ] Home to OPEN limit at reduced speed on every boot
  - [ ] SNTP validity check with 30s timeout
  - [ ] Post-homing schedule evaluation (close if in window and time confirmed)
  - [ ] Fail-open if time not confirmed
- [ ] Door state machine:
  - [ ] Open and close sequences with step counting
  - [ ] Limit switch interrupt handlers (unexpected trip → ERROR + HA alert)
  - [ ] Door state text entity synced to HA (open/closed/moving/homing/error)
  - [ ] ERROR state: accept manual commands; require HA clear to resume automation
- [ ] Scheduled door (time component trigger → state machine)
- [ ] Manual door open/close (HA buttons; neither cancels schedule)
- [ ] Food low alert (HC-SR04 threshold → HA notification)
- [ ] Water low alert (XKC-Y25 → HA notification)
- [ ] Egg count:
  - [ ] CW/CCW ±1 (floor 0, ceiling 99)
  - [ ] Push-hold 2s reset with debounce
  - [ ] NVS persistence (egg count only)
  - [ ] TM1637 display update
  - [ ] HA number sync
- [ ] Nest box occupancy (LD2410 → HA binary sensor)
- [ ] LED status logic (open, moving, alert, error states)
- [ ] Relay control from HA
- [ ] OTA update confirmed
- [ ] Code review and lambda documentation

**Checkpoint 5 Exit Criteria:**
> All v1 features work end-to-end. Boot sequence, homing, SNTP check, and post-homing schedule evaluation all tested. ERROR state and manual recovery tested. No critical bugs.

---

### 🔲 Phase 6 — Home Assistant Integration & Automations

**Tasks:**
- [ ] HA dashboard (door state, food/water, egg count, temp/humidity, nest occupancy)
- [ ] Door schedule automations (open and close)
- [ ] Error alert automation (door error → mobile push notification)
- [ ] Food/water alert automations
- [ ] Midnight egg count reset
- [ ] Nest occupancy log (placeholder for v2 camera trigger)
- [ ] Test all automations including power-loss recovery scenario
- [ ] Logbook entries for door events and daily egg totals

**Checkpoint 6 Exit Criteria:**
> Dashboard and automations complete. Power-loss scenario tested end-to-end.

---

### 🔲 Phase 7 — Enclosure Build & Final Assembly

**Tasks:**
- [ ] Install PCB into Hoffman A806CH
- [ ] Mount NEMA 17 at coop frame; install SK8 bearing block at far end
- [ ] Install drawer slide; attach TR8 nut bracket to moving member / door panel
- [ ] Mount limit switches on fixed member; attach actuating tabs to moving member
- [ ] Route all cables to fixed frame — no flex wiring to door panel
- [ ] Mount TM1637 and encoder on A806CH faceplate
- [ ] Install HC-SR04 in hopper, water sensor, LED strip
- [ ] Verify all functions after final assembly
- [ ] 24-hour soak test (door cycles, boot/homing, sensors, HA monitoring)

**Checkpoint 7 Exit Criteria:**
> Device operates in enclosure. 24-hour soak test passes. No flex wiring to moving parts.

---

### 🔲 Phase 8 — Testing, Validation & Documentation

**Tasks:**
- [ ] Hardware docs (schematic PDF, BOM, wiring, door mechanism photos)
- [ ] Firmware docs (YAML with comments, state machine diagram, config guide)
- [ ] HA setup guide (device add, automation templates, error recovery procedure)
- [ ] Document known limitations
- [ ] Photograph final build
- [ ] Final git commit / backup all source files
- [ ] Define and prioritize v2 backlog

**Checkpoint 8 Exit Criteria:**
> Documentation complete. Device in daily use. v2 backlog defined.

---

## 6. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Drawer slide binds or corrodes outdoors | Medium | High | Stainless or zinc slide; lubricate at soak test and periodically |
| Soft-close detent on slide fights stepper | High if wrong slide | Medium | Explicitly spec no-soft-close in Phase 1 BOM |
| Door panel misaligned on slide (racking) | Medium | Medium | TR8 nut and slide moving member both rigid to same panel |
| SNTP unavailable for extended period after boot | Low | Low | Fail-open; HA sends alert; schedule closes when time syncs |
| Homing move during nighttime power blip briefly opens door | Low | Low | Accepted tradeoff vs. complexity of NVS state restore |
| HC-SR04 unreliable in dusty hopper | Medium | Medium | 5-sample median filter; test in real conditions in Phase 4 |
| LD2410 false positives in nest box | Medium | Low | Tune sensitivity zone; false positives don't trigger actuators |
| NEMA 17 stall on jammed door | Low | Medium | TMC2208 stall guard detection; ERROR state on unexpected limit trip |
| Wi-Fi dropout causes missed schedule | Low | High | ESPHome time caches schedule locally; operates offline |
| Hoffman A806CH too tight | Medium | Medium | Validate in Phase 2; A1008CH fallback |
| 12V noise affecting sensor readings | Medium | Medium | Decouple all rails; separate motor and sensor traces in Phase 3 |

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
| — | MCU: ESP32-S3 | GPIO count, ESPHome, camera-ready for v2 | Locked |
| — | Firmware: ESPHome | HA native, OTA, mature sensor support | Locked |
| — | Door orientation: horizontal sliding | Space constraints at installation | Locked |
| — | Door guide: 16" full-extension no-soft-close slide | Off-the-shelf; no soft-close avoids detent interference | Locked |
| — | Door drive: TR8 + NEMA 17 48mm (17HS8401) + TMC2208 | Self-locking; 65 N·cm torque headroom; silent | Locked |
| — | Homing: always to OPEN limit on every boot | Unconditional fail-open; no NVS door state | Locked |
| — | Boot behavior: home → SNTP check (30s) → evaluate schedule | Time-aware close after safe open; fails open if no time | Locked |
| — | No door state written to NVS | Simplicity; homing is always the recovery path | Locked |
| — | Manual close/open: does not modify schedule | Predictable; schedule always resumes | Locked |
| — | Door geometry: 12"×12" opening, 2" far lip, several feet motor side | Confirmed installation dimensions | Locked |
| — | Display: TM1637 7-segment 2-digit | 2 GPIOs; ESPHome native | Locked |
| — | Local input: rotary encoder only | 3 GPIOs; keypad to v2 | Locked |
| — | Occupancy: LD2410 mmWave | Detects stationary hen; PIR misses still animals | Locked |
| — | Food level: HC-SR04 | On hand; median filter for dust | Locked |
| — | Power: 12V main + LM2596 5V buck + AMS1117 3.3V | Single wall cable; all rails internal | Locked |
| — | Enclosure: Hoffman A806CH (6"×8") | Must fit; A1008CH fallback | Locked |
| — | Camera: GPIO reserved, header unpopulated in v1 | v2 egg detection path | Locked |

---

*Document version 1.4 — Boot behavior locked (home-to-open → SNTP check → schedule evaluate). No NVS door state. Door geometry confirmed (12"×12", 16" slide). Power architecture locked (12V + LM2596 buck). All Phase 0 decisions resolved.*