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
| F5 | **Local display** | SSD1306 OLED (128×64, I2C) — shows egg count, temp/humidity, door state; extensible for menus |
| F6 | **Local input — egg count** | Rotary encoder: turn = ±1, push-hold 2s = reset to 0; co-located with display in remote 3D-printed box |
| F7 | **Nest box occupancy** | LD2410 mmWave — detects stationary hen in nest box |
| F8 | **Temperature & humidity** | Environmental data reported to HA |
| F9 | **Status LEDs** | WS2812B strip for power, door state, alert states |
| F10 | **Door drive — horizontal leadscrew** | NEMA 17 48mm + TR8 leadscrew + TMC2208; door rides on heavy-duty drawer slide |
| F11 | **Relay output** | Switch a connected load (water heater/de-icer, pump, heat lamp, etc.) |
| F12 | **Wi-Fi OTA firmware updates** | Reflash without physical access |
| F13 | **Power loss recovery** | Fail-open by design; time-aware post-homing schedule check |
| F14 | **Egg production analytics** | InfluxDB time-series storage + Grafana dashboards; multi-year history; historical import supported |

### v2 — Future (Scoped Out of v1)

- Camera feed + AI egg detection (automatic egg count logging)
- Power/energy monitoring of relay load
- Food weight measurement (load cell + HX711)
- Automatic water top-up (second relay + peristaltic pump)
- Full menu/navigation UI via rotary encoder + SSD1306 (foundation already present in v1)
- 4x4 keypad for direct numeric egg count entry
- Full TFT display (ILI9341) if SSD1306 proves limiting (unlikely)

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
**Door panel:** Lightweight wood — hardboard or thin plywood with a simple frame. Full seal not required; goal is predator exclusion and snow/wind blocking.

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
| Candidate | https://a.co/d/02JIuqZc |

#### Limit Switch Mounting

Both switches mount to the **fixed member** (wall/frame side). A passive actuating tab on the **moving member** (door side) trips each switch. All wiring is stationary — no flex wiring to the moving door.

| Switch | Position | Function |
|--------|----------|----------|
| OPEN limit | Fixed member, fully-retracted position | Homing reference: trips when door fully open → zero step count |
| CLOSED limit | Fixed member, fully-extended position | Safety cutoff at full-closed travel; also confirms closed state |

**Limit switch:** V-153-1C25 (on hand)

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

### 3.3 Component Placement Philosophy

Most sensors and user-interface components are **not** mounted inside the Hoffman enclosure. 3D-printed boxes/mounts will be designed and printed for each peripheral, placed in their optimal location:

| Component | Location | Enclosure |
|-----------|----------|-----------|
| ESP32-S3, stepper driver, relays, power | Hoffman A806CH | Main enclosure, fixed to coop wall |
| SSD1306 display + rotary encoder | Accessible coop wall or interior panel | 3D-printed remote UI box |
| LD2410 mmWave | Near/above nest box | 3D-printed mount |
| SHT31 temp/humidity | Coop interior, shaded from direct sunlight | 3D-printed vented mount |
| HC-SR04 food level | Inside/above hopper | 3D-printed angled mount |
| XKC-Y25 water level | Mounted on water reservoir wall | Direct adhesive or 3D-printed bracket |
| WS2812B LED strip | Coop exterior or visible interior location | Exposed or weather-resistant channel |
| Limit switches | Fixed side of drawer slide frame | Direct mount to frame |

> **I2C cable run note:** The SSD1306 and SHT31 share I2C Bus 0. The SSD1306 is in a remote box, meaning the I2C bus runs over an external cable. Standard I2C (4.7 kΩ pull-ups) is reliable to ~1 meter. If the remote display box is further than ~1 meter from the Hoffman enclosure, reduce pull-ups to 2.2 kΩ or consider an I2C extender (P82B96 or similar). **Measure planned cable run in Phase 2 and select pull-up values accordingly.**

---

### 3.4 Peripheral Component Selections

#### Sensors & Inputs

| Component | Part | Bus / Interface | Notes |
|-----------|------|-----------------|-------|
| Temp / Humidity | SHT31 or AHT21 | I2C (Bus 0, addr 0x44) | Stable in outdoor/coop environment; in vented 3D-printed mount |
| Food level | HC-SR04 (on hand) | GPIO trigger/echo | Distance from sensor to food surface; 5-sample median filter |
| Water level | XKC-Y25 capacitive | GPIO (digital) | No exposed electrodes |
| Nest box occupancy | LD2410 mmWave | UART | Detects stationary hen; in 3D-printed mount near nest box |
| Door limit (open) | V-153-1C25 roller microswitch | GPIO (digital in, pull-up) | Homing reference; zero at full open |
| Door limit (closed) | V-153-1C25 roller microswitch | GPIO (digital in, pull-up) | Safety cutoff + closed confirmation |
| Rotary encoder | EC11 with push button | GPIO (A, B, SW) | In remote UI box with display; egg count ±1; push-hold 2s = reset |

#### Outputs & Actuators

| Component | Part | Interface | Notes |
|-----------|------|-----------|-------|
| OLED display | SSD1306 128×64 I2C module | I2C (Bus 0, addr 0x3C) | In remote 3D-printed UI box with encoder; egg count, temp, door state |
| Door stepper | NEMA 17 48mm, 17HS8401 | STEP / DIR / EN | Horizontal door via TR8 |
| Stepper driver | TMC2208 | STEP / DIR / EN | Silent; 12V supply |
| Relay | 5V single-channel relay module | GPIO | Water heater/de-icer, pump, or other load |
| LED strip | WS2812B addressable | GPIO (RMT) | Door state and alert status |
| Buzzer (optional) | Passive piezo | GPIO (PWM) | May be omitted |

#### Power

| Rail | Source | Consumers |
|------|--------|-----------|
| 12V (main) | 12V 2A wall adapter | NEMA 17 via TMC2208 |
| 5V | LM2596 buck converter from 12V main | Relay coil, logic |
| 3.3V | AMS1117-3.3 LDO from 5V | ESP32-S3, sensors, encoder, SSD1306 |

> **Power architecture locked:** Single 12V main supply with LM2596 5V buck. One wall cable to enclosure; all rails derived internally. Simplifies enclosure entry and wiring.

---

### 3.5 Bus / GPIO Map (Preliminary)

| Bus | Devices |
|-----|---------|
| I2C (Bus 0) | SHT31 temp/humidity (0x44) + SSD1306 display (0x3C) — shared bus, remote cable run, see pull-up note in §3.3 |
| UART 0 | LD2410 mmWave |
| UART 1 | Reserved for v2 camera / debug |
| GPIO digital in | HC-SR04 trigger/echo, XKC-Y25, door limits ×2, encoder A/B/SW |
| GPIO digital out | Relay, stepper STEP/DIR/EN, buzzer |
| GPIO RMT | WS2812B LED strip |
| Camera DVP header | Reserved, unpopulated in v1 |

> **I2C note:** SSD1306 at 0x3C and SHT31 at 0x44 — no address conflict. SSD1306 is remote over external cable; keep pull-ups at 4.7 kΩ for runs ≤1m; reduce to 2.2 kΩ or add P82B96 extender for longer runs. Measure actual cable run in Phase 2.

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
| OLED display | `display: platform: ssd1306_i2c` — 128×64; pages for egg count, env data, door state |
| Rotary encoder | `rotary_encoder:` component |
| NVS persistence | `globals: restore_value: true` — egg count only; **no door state written to NVS** |
| Time | `time: platform: sntp` with validity check in lambda |
| Custom logic | Lambda C++ for door state machine, homing routine, boot sequence |

### 4.2 Analytics Stack — InfluxDB + Grafana

Egg production history is stored in InfluxDB (time-series database) and visualized in Grafana. Both run as Home Assistant add-ons — no separate server required.

**Data flow:**

```
ESP32-S3 (ESPHome)
  └── number.coop_egg_count → HA entity
        └── HA influxdb: integration → InfluxDB (time-series DB)
              └── Grafana queries InfluxDB → production dashboards
```

**Recording strategy (Option 1 — derive from counter):**  
InfluxDB records every state change of `number.coop_egg_count`. The daily egg total is derived at query time as `max(egg_count)` for each calendar day, since the counter only increases during the day and resets to 0 at midnight. No additional HA entities or firmware changes are required.

**Retention:** Configured for indefinite retention — suitable for multi-year production tracking.

**Historical import:** Prior egg count records can be imported via CSV using the InfluxDB CLI or a short Python script against the InfluxDB HTTP API. See `InfluxDB_Grafana_Setup.md` for the import procedure.

#### Target Grafana Dashboards

| Dashboard | Description |
|-----------|-------------|
| Daily production | Eggs per day, rolling 30/90 days |
| Weekday breakdown | Average eggs by day of week (Mon–Sun) |
| Monthly totals | Bar chart, year-over-year comparison |
| Yearly curve | Seasonal production trend — identifies winter slowdown |
| All-time record | Cumulative total and personal-best day |

### 4.3 Home Assistant Integration Points

| HA Entity | Type | Source |
|-----------|------|--------|
| `sensor.coop_temperature` | Sensor | SHT31 |
| `sensor.coop_humidity` | Sensor | SHT31 |
| `sensor.coop_food_distance_cm` | Sensor | HC-SR04 |
| `binary_sensor.coop_food_low` | Binary Sensor | HC-SR04 threshold |
| `binary_sensor.coop_water_low` | Binary Sensor | XKC-Y25 |
| `binary_sensor.coop_nest_occupied` | Binary Sensor | LD2410 |
| `number.coop_egg_count` | Number | Rotary encoder, synced to HA; forwarded to InfluxDB |
| `button.coop_door_open` | Button | Manual open command |
| `button.coop_door_close` | Button | Manual close (schedule resumes unchanged) |
| `sensor.coop_door_state` | Text Sensor | open / closed / moving / homing / error |
| `switch.coop_relay` | Switch | Controls relay output |
| `light.coop_leds` | Light | WS2812B strip |

### 4.4 Door State Machine

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

### 4.5 Egg Count Logic

```
Rotary encoder CW        → egg_count + 1 (max 99)
Rotary encoder CCW       → egg_count - 1 (min 0)
Push-hold 2 seconds      → egg_count reset to 0 (debounced)
On any change            → update SSD1306 display + sync number.coop_egg_count to HA
                           → HA influxdb: integration forwards change to InfluxDB
Midnight (HA automation) → number.set_value resets to 0
Power-on restore         → globals restore_value from NVS (egg count only)
```

### 4.5 Display Layout (SSD1306 128×64)

Default display page (always-on or rotating):

```
┌────────────────────────┐
│  EGGS: 07              │  ← Large font, primary info
│  DOOR: CLOSED          │  ← Door state text
│  72°F  58%RH           │  ← Temp + humidity
│  [ALERT: FOOD LOW]     │  ← Alert line (blank when no alert)
└────────────────────────┘
```

> v2 menu navigation (rotary encoder short-press = cycle pages, long-press = select) is a natural extension of the existing encoder and display — no hardware changes required.

---

## 5. Phase Breakdown & Checkpoints

### ✅ Phase 0 — Project Foundation

**Locked:**
- [x] MCU: ESP32-S3, Firmware: ESPHome
- [x] Display: SSD1306 OLED 128×64, I2C, remote 3D-printed UI box with rotary encoder
- [x] Local input: rotary encoder only
- [x] Food level: HC-SR04 (on hand)
- [x] Door orientation: horizontal sliding
- [x] Door guide: heavy-duty full-extension drawer slide, 16" travel, no soft-close
- [x] Door opening: 12" × 12"; far-side lip 2"; motor-side clearance several feet
- [x] Door panel: lightweight wood (hardboard or thin plywood + simple frame); full seal not required
- [x] Door drive: TR8 leadscrew + NEMA 17 48mm (17HS8401) + TMC2208
- [x] Homing: always to OPEN limit on boot; zero at open
- [x] Boot behavior: home to OPEN → SNTP check (30s timeout) → close if in schedule window; else stay open
- [x] No door state written to NVS
- [x] Manual close: does not cancel open schedule
- [x] Occupancy: LD2410 mmWave
- [x] Power: 12V main + LM2596 5V buck; 3.3V LDO from 5V
- [x] Enclosure: Hoffman A806CH (6"×8"); A1008CH fallback (main electronics only)
- [x] Camera GPIO/header reserved for v2
- [x] Sensor placement: 3D-printed mounts/boxes; most sensors external to Hoffman enclosure
- [x] Analytics: InfluxDB + Grafana (HA add-ons); Option 1 daily total derivation; indefinite retention

**Open:**
- [ ] Drawer slide specific part number and source — **Phase 1**
- [ ] Door panel material — **Phase 2**
- [ ] SSD1306 remote cable run length and pull-up confirmation — **Phase 2**

**Checkpoint 0 Exit Criteria:**
> All hardware, firmware, and behavior decisions documented and locked. No ambiguity on v1 scope.

---

### 🔲 Phase 1 — Component Selection & BOM

**Tasks:**
- [ ] Source TR8 leadscrew (≥350mm), NEMA 17 17HS8401, TMC2208, flexible coupler, SK8 bearing block
- [ ] Confirm and order 16" full-extension no-soft-close slide (stainless or zinc, 100 lb+):
    - Candidate: https://a.co/d/02JIuqZc — confirm no soft-close detent before ordering
- [ ] Confirm limit switches: V-153-1C25 (on hand)
- [ ] Select SSD1306 128×64 I2C module (3.3V compatible; 4-pin I2C breakout preferred)
- [ ] Select EC11 rotary encoder with push button
- [ ] Select LD2410 mmWave module
- [ ] Select LM2596 5V buck module
- [ ] Select SHT31 (or AHT21) temp/humidity breakout
- [ ] Create full BOM: part, qty, supplier, unit cost, link
- [ ] Calculate power budget (mA per rail, worst case with 1.2A RMS motor + all peripherals)
- [ ] Confirm 12V 2A supply is adequate (motor stall + all logic + SSD1306 ~20mA)
- [ ] Identify long-lead parts; order early
- [ ] Order all components

**Checkpoint 1 Exit Criteria:**
> BOM complete. All parts ordered. Power budget confirms 12V 2A is adequate.

---

### 🔲 Phase 2 — Mechanical & Enclosure Design

**Tasks:**
- [ ] Fabricate door panel: lightweight wood (hardboard or thin plywood + simple framed border), 13–14" wide × ~13" tall
- [ ] Plan motor mount at one end of coop frame; leadscrew parallel to slide
- [ ] Plan SK8 bearing block position at far end of leadscrew
- [ ] Attach TR8 nut bracket to door panel / slide moving member
- [ ] Plan limit switch positions on fixed member; fabricate actuating tabs on moving member
- [ ] Verify door clears opening fully when open; verify 2" lip overlap when closed
- [ ] Design and print 3D-printed remote UI box (SSD1306 + rotary encoder); route I2C + encoder wires back to Hoffman enclosure
- [ ] **Measure I2C cable run from Hoffman enclosure to remote UI box; confirm pull-up resistor values (≤2.2 kΩ if >1 meter)**
- [ ] Design and print LD2410 mount near nest box; plan UART cable routing
- [ ] Design and print vented SHT31 mount (shaded, not in direct sun or heat)
- [ ] Design and print HC-SR04 mount angle inside hopper
- [ ] Plan XKC-Y25 mount on water reservoir
- [ ] Plan TM1637 breakout on A806CH faceplate (replaced by external display — confirm only encoder cable entry on faceplate)
- [ ] Reserve camera position on Hoffman faceplate (v2)
- [ ] Plan cable routing: all door wiring to fixed frame; no flex wiring to door panel

**Checkpoint 2 Exit Criteria:**
> Door mechanism dimensionally validated. All 3D-printed mounts designed. I2C cable run measured and pull-up values confirmed. No cable routing to moving parts.

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
- [ ] Define connectors: JST-PH for sensors/display, screw terminals for motor/power
- [ ] Decide: perfboard vs. custom PCB

**Checkpoint 3 Exit Criteria:**
> Schematic complete and reviewed. GPIO map locked. Power budget validated in schematic. I2C pull-ups confirmed.

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
  - [ ] SSD1306 OLED over remote cable
- [ ] Test each output individually:
  - [ ] SSD1306: render egg count, door state, temp/humidity page; confirm readable at cable run distance
  - [ ] NEMA 17 + TMC2208: rotate, reverse, homing to OPEN limit
  - [ ] Relay on/off
  - [ ] WS2812B color test
- [ ] Wire all subsystems; confirm no conflicts
- [ ] Full ESPHome config — all entities visible in HA
- [ ] Confirm `number.coop_egg_count` changes appear in InfluxDB

**Checkpoint 4 Exit Criteria:**
> All subsystems functional in HA. Homing to OPEN limit works on bench. No pin conflicts. SSD1306 renders correctly over full cable run. InfluxDB receiving egg count data.

---

### 🔲 Phase 5 — Firmware Development

**Tasks:**
- [ ] Boot / homing sequence (Section 4.4):
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
  - [ ] SSD1306 display update (egg count + door state)
  - [ ] HA number sync → InfluxDB forwarding via HA integration
- [ ] SSD1306 display pages (egg count, env data, door state, alert line)
- [ ] Nest box occupancy (LD2410 → HA binary sensor)
- [ ] LED status logic (open, moving, alert, error states)
- [ ] Relay control from HA
- [ ] OTA update confirmed
- [ ] Code review and lambda documentation

**Checkpoint 5 Exit Criteria:**
> All v1 features work end-to-end. Boot sequence, homing, SNTP check, and post-homing schedule evaluation all tested. ERROR state and manual recovery tested.SSD1306 display pages correct. No critical bugs.

---

### 🔲 Phase 6 — Home Assistant Integration & Automations

**Tasks:**
- [ ] Install InfluxDB add-on; create `chickmonitor` database with infinite retention policy
- [ ] Configure HA `influxdb:` integration to forward `number.coop_egg_count` to InfluxDB
- [ ] Install Grafana add-on; add InfluxDB as data source
- [ ] Build Grafana dashboards (see Section 4.2 target list)
- [ ] Verify daily total derivation: `max(egg_count)` per calendar day returns correct values
- [ ] HA dashboard (door state, food/water, egg count, temp/humidity, nest occupancy)
- [ ] Door schedule automations (open and close)
- [ ] Error alert automation (door error → mobile push notification)
- [ ] Food/water alert automations
- [ ] Midnight egg count reset automation
- [ ] Nest occupancy log (placeholder for v2 camera trigger)
- [ ] Test all automations including power-loss recovery scenario
- [ ] Logbook entries for door events and daily egg totals

**Checkpoint 6 Exit Criteria:**
> Dashboard and automations complete. InfluxDB receiving data and Grafana dashboards rendering correctly. Power-loss scenario tested end-to-end.

---

### 🔲 Phase 7 — Enclosure Build & Final Assembly

**Tasks:**
- [ ] Install PCB into Hoffman A806CH
- [ ] Mount NEMA 17 at coop frame; install SK8 bearing block at far end
- [ ] Install drawer slide; attach TR8 nut bracket to moving member / door panel
- [ ] Mount limit switches on fixed member; attach actuating tabs to moving member
- [ ] Route all cables to fixed frame — no flex wiring to door panel
- [ ] Install 3D-printed remote UI box (SSD1306 + encoder); dress I2C and encoder cable
- [ ] Install 3D-printed LD2410 mount near nest box; route UART cable
- [ ] Install HC-SR04 in hopper, water sensor, SHT31 in vented mount, LED strip
- [ ] Verify all functions after final assembly
- [ ] 24-hour soak test (door cycles, boot/homing, sensors, HA monitoring, InfluxDB data continuity)

**Checkpoint 7 Exit Criteria:**
> Device operates in enclosure. 24-hour soak test passes. No flex wiring to moving parts. InfluxDB logging continuously.

---

### 🔲 Phase 8 — Testing, Validation & Documentation

**Tasks:**
- [ ] Hardware docs (schematic PDF, BOM, wiring, door mechanism photos)
- [ ] Firmware docs (YAML with comments, state machine diagram, config guide)
- [ ] HA setup guide (device add, automation templates, error recovery procedure)
- [ ] Import historical egg count data into InfluxDB (see `InfluxDB_Grafana_Setup.md`)
- [ ] Validate historical data appears correctly in all Grafana dashboards
- [ ] Document known limitations
- [ ] Photograph final build
- [ ] Final git commit / backup all source files
- [ ] Define and prioritize v2 backlog

**Checkpoint 8 Exit Criteria:**
> Documentation complete. Historical data imported and verified. Device in daily use. v2 backlog defined.

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
| I2C signal degradation over long cable run to SSD1306 | Medium | Medium | Measure cable run in Phase 2; reduce pull-ups to ≤2.2 kΩ if >1 meter; use twisted pair or shielded cable |
| Lightweight wood door warps in moisture/cold | Low | Low | Simple frame stiffens panel; no seal required; monitor after installation |
| InfluxDB data loss on HA host failure | Low | Medium | Periodic InfluxDB backup; HA snapshot includes add-on data |

---

## 7. v2 Backlog

| Priority | Feature | Notes |
|----------|---------|-------|
| High | **Camera + AI egg detection** | ESP32-S3 camera interface reserved; auto egg count logging |
| High | **Power / energy monitoring** | INA219 or PZEM-004T on relay load |
| Medium | **Food weight (load cell)** | HX711 + load cell; more precise than distance sensor |
| Medium | **Automatic water top-up** | Second relay + peristaltic pump |
| Medium | **Rotary encoder menu navigation** | SSD1306 hardware already present in v1; extend display firmware only |
| Low | **TFT display (ILI9341)** | Only if SSD1306 128×64 proves limiting — unlikely |
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
| — | Display: SSD1306 OLED 128×64 I2C | More info than 7-seg; I2C shares Bus 0 with SHT31; ESPHome native; extensible for v2 menus | Locked |
| — | Display location: remote 3D-printed UI box with rotary encoder | Optimal user access; keep Hoffman enclosure clean | Locked |
| — | Local input: rotary encoder only | 3 GPIOs; keypad to v2 | Locked |
| — | Occupancy: LD2410 mmWave | Detects stationary hen; PIR misses still animals | Locked |
| — | Food level: HC-SR04 | On hand; median filter for dust | Locked |
| — | Power: 12V main + LM2596 5V buck + AMS1117 3.3V | Single wall cable; all rails internal | Locked |
| — | Enclosure: Hoffman A806CH (6"×8") | Main electronics only; A1008CH fallback | Locked |
| — | Camera: GPIO reserved, header unpopulated in v1 | v2 egg detection path | Locked |
| — | Door panel: lightweight wood (hardboard or thin plywood + frame) | Easy to fabricate; full seal not required; predator and snow exclusion only | Locked |
| — | Sensor placement: 3D-printed external mounts | Optimal sensor positioning; keeps Hoffman enclosure uncluttered | Locked |
| — | Limit switches: V-153-1C25 (on hand) | Standard roller microswitch; proven reliable | Locked |
| — | Egg analytics: InfluxDB + Grafana as HA add-ons | Time-series DB purpose-built for this workload; local; no cloud | Locked |
| — | Egg daily total: Option 1 — max(egg_count) per day at query time | No extra entities or firmware; counter only increases intra-day | Locked |
| — | InfluxDB retention: indefinite | Multi-year production history required | Locked |

---

*Document version 1.6 — Added egg production analytics (F14): InfluxDB + Grafana as HA add-ons, Option 1 daily total derivation, indefinite retention, historical import. Updated display from TM1637 to SSD1306 OLED in remote 3D-printed UI box. I2C bus map updated (SSD1306 0x3C, SHT31 0x44). Phase 6 and Phase 8 tasks updated. Risk register and decisions log updated.*