# ROAM Edge — Technical Specification

**Project:** `roam_edge_xl` (KiCad 7/8 hardware design)
**Document type:** Hardware technical specification
**Prepared:** 4 September 2026
**Status:** Design in progress — layout complete, DRC clean-up outstanding (see §10)

---

## 1. Overview

ROAM Edge is a vehicle-mounted **edge telematics and control unit** for electric two-
and three-wheelers (bodaboda-class fleets). It combines an application processor with
cellular backhaul, satellite positioning, motion sensing, a CAN/RS-485 vehicle interface
and opto-isolated ignition/immobiliser control on a single 6-layer board that installs
inline on the vehicle harness.

Functionally the unit does four things: it **locates** the vehicle (GNSS), **reports**
telemetry over the mobile network (LTE), **listens to and talks on the vehicle bus**
(CAN 500 kbit/s and RS-485), and **acts on the vehicle** through relay/immobiliser
outputs while sensing ignition, tamper and lid state. On-board flash buffers telemetry
so the unit survives cellular dead-zones, and a backup-battery charger keeps it alive
when main power is cut.

The design is realised as a **family of board variants** — see §2 — sharing one core
architecture (ESP32-S3 + Quectel modem + GNSS + CAN). This document specifies the core
architecture and then describes how each variant differs.

---

## 2. The "Edge" family — variant map

The repository is named for the flagship **Edge XL** board but contains several related
"Edge" designs and build variants. Every instance of *Edge* in the project is catalogued
here so the naming is unambiguous.

| Instance | Where it appears | What it is |
|---|---|---|
| **`roam_edge_xl`** | `roam_edge_xl.kicad_pro/.kicad_sch/.kicad_pcb` | The flagship "Edge XL" board — the primary, fully-routed design. Full feature set: LTE Cat 1, GNSS, CAN, RS-485, boost rail, level shifters. |
| **`EDGE_A42C`** | `EDGE_A42C.step`, `EDGE_A42C_BOM_DNP(.csv)` | The **Edge XL assembly build "A42C"** — a specific manufacturing revision of `roam_edge_xl` with its own do-not-place (DNP) list and 3-D export. This is the build most of the BOM data describes. |
| **`roam_edge+` / `EDGE_+`** | `roam_edge+.kicad_sch`, `EDGE_+.step` | The **"Edge Plus"** variant. Same ESP32-S3 core; carries alternate modem placements (`EG915U-EU`, `EG915N-N06-SGNSA`, generic `EG91xQ`) and the boost converter. A sibling to XL rather than a subset. |
| **`LITE_E4C`** | `LITE_E4C_BOM` | The **"Edge Lite" build "E4C"** — a cost-reduced variant. Swaps the modem for a Quectel **EG800K**, uses a different power chain and an 11-pin JST harness, and drops several XL-only front-ends. |
| **`EDGE:` footprint library** | `EDGE:CA-L01`, `EDGE:XCVR_LC79H`, `EDGE:TPS566242DRLR` | Project-local KiCad footprint library named `EDGE`, holding the custom footprints (cellular antenna, GNSS module, buck regulator) shared across the boards. Not a board — a library namespace. |
| **`flasher_relay`** | `flasher_relay/` (own `.kicad_pro/.sch/.pcb`) | A small **companion relay/flasher board**, a separate KiCad project. Drives the vehicle's flasher/immobiliser high-current loads from the Edge logic outputs (`T6_OUT`, `IM_OUT`, `Flasher_Out`). |

> **Naming summary:** `XL` and `+` (Plus) are two board designs; `A42C`, `E4C` and the
> `_BOM_DNP` files are **build/assembly variants** (population options) of those boards;
> `EDGE:` is a footprint library; `flasher_relay` is a companion board. The rest of this
> spec describes the **Edge XL (`roam_edge_xl`, build A42C)** unless a variant is named.

---

## 3. Physical & board specification (Edge XL)

| Parameter | Value |
|---|---|
| PCB technology | 6-layer rigid FR-4 |
| Board thickness | 1.65 mm |
| Outline | ≈ **103.0 mm × 60.0 mm** (rectangular, from Edge.Cuts extent) |
| Footprints placed | 230 (119 top / 88 bottom populated positions) |
| Nets | 1,095 |
| Assembly | Double-sided SMT, mixed 0402/0603/0805 passives, QFN/SON/LGA actives |
| Fabrication toolkit | JLCPCB "Fabrication Toolkit" options file present (`fabrication-toolkit-options.json`) |

**Layer stack-up (top → bottom):**

| Layer | Name | Role |
|---|---|---|
| F.Cu | TOP | Signal + component |
| In1.Cu | GND02 | Ground plane |
| In2.Cu | PWR03 | Power distribution |
| In3.Cu | Signal | Mixed signal (inner routing) |
| In4.Cu | GND01 | Ground / power plane |
| B.Cu | BOTTOM | Signal + component |

The stack pairs each outer signal layer against a ground reference (F.Cu/GND02,
B.Cu/GND01) for controlled-impedance RF routing (cellular and GNSS antenna feeds),
with a dedicated power layer for the multi-rail supply.

**Design rules (from project settings):** min track 0.20 mm, min clearance 0.10 mm,
min via Ø 0.25 mm, min hole-to-hole 0.20 mm, min annular ring 0.075 mm, microvias
enabled (min Ø 0.20 mm / drill 0.10 mm). Differential-pair rules are defined for the
USB and RF pairs.

---

## 4. System architecture

```
        VEHICLE HARNESS  (J8, 18-pin 2.5 mm MX23A18NF1)
   V_EXT ─┬─ CAN_H/L ── T6_OUT ── IM_OUT ── Flasher_Out ── IGN ── TAMPER ── D±/AB ── GND
          │
   [Protection: F1 polyfuse · CR1 SMCJ100A TVS(100V) · CR2 PLZ15C zener · Q9 AO4407A
    reverse-polarity P-FET · U9 SQS160 load switch]
          │  VIN
          ├──► U3 TPS566242 (6A buck) ─► 5V0
          ├──► U4 EG1192L (3A buck) ─► 4V0  (modem)
          ├──► U18 TPS61022 (boost) ─► 4V7_BOOST      [XL/Plus]
          └──► U10/U21/U27 LP5907-3.3 LDO ×3 ─► 3V3_ALWAYS_ON · 3V3_LDO(switched) · 3V3_ANT
                                        │
   U6 W25Q128 ◄── QSPI ──┐             │  (3V3_LDO_EN gates the peripheral rail)
                          ▼             ▼
                   ┌──────────────────────────┐
   USB (J4)◄─NUP4202─►│   U25 ESP32-S3-MINI-1  │◄── I²C/SPI ── U13 LIS2DH12 accelerometer
                       │   (Wi-Fi/BLE + TWAI)   │
   U26 TJA1051 ◄─CAN──►│                        │──ADC── IBL (battery divider)
   (500 kbit/s)        │                        │──GPIO── STATUS/CONN/NRF LEDs
   U7 EG915U-EU ◄UART─►│                        │──GPIO── T6_SIG / IM_OUT_3V3 (control)
   LTE Cat 1           └──────────┬─────────────┘
   ├ SIM (SIM8060)                │ UART
   └ ANT CA-L01 (J6)     U20 LC760Z GNSS ── IPEX ANT (J3)
                         (GPS/GLONASS/BDS/Galileo/QZSS)
   U11 TXB0108 level shift ─► RS-485 (U22 MAX13487) ─► RS485_TX/RX_5V
   U5/U19 LTV-217 opto ─► IGN_IN / isolated inputs
   TP4056 ─► backup-battery charge
```

At the centre is the **ESP32-S3** application processor. Every radio and bus hangs off
it: the cellular modem and GNSS over UART, the accelerometer over I²C/SPI, CAN through
the ESP32's built-in TWAI controller and an external transceiver, and RS-485 through a
level shifter and half-duplex transceiver. A switched 3.3 V rail (`3V3_LDO_EN`) lets
firmware power-gate the transceivers, sense front-ends and LEDs to save energy when the
vehicle is parked.

---

## 5. Subsystem specifications

### 5.1 Compute & storage
- **U25 — Espressif ESP32-S3-MINI-1-N8.** Dual-core Xtensa LX7, Wi-Fi 802.11 b/g/n,
  Bluetooth LE, 8 MB in-package flash, integrated PCB antenna. Runs the application
  firmware and hosts the **TWAI (CAN)** controller natively.
- **U6 — Winbond W25Q128JVP.** 128 Mbit / 16 MiB QSPI NOR flash (WSON-8) for telemetry
  store-and-forward, firmware images and configuration.

### 5.2 Cellular (LTE)
- **U7 — Quectel EG915U-EU** (`EG915UEUAC-N05-SNNSA`), LTE **Cat 1** modem, UART-attached
  to the ESP32 (`MODEM_TX/RX_3V3`, `RTS/CTS`, `DTR`, `RI`, `PWRKEY`). Powered from the
  **4V0** rail (EG1192L buck).
- **SIM:** GCT **SIM8060** push-push nano-SIM socket, with dedicated `SIM_VDD` and a
  SIM-side ESD/level interface. (`USIM2` on the modem is unused.)
- **Antenna:** **CA-L01** LTE/GSM SMD antenna at **J6**, on a dedicated matched feed
  (`3V3_ANT` bias network with L/C matching parts L2–L6).
- **Variant modems:** Edge Plus places alternates (`EG915N-N06-SGNSA`, generic `EG91xQ`);
  Edge Lite uses the **EG800K** (`XCVR_EG800Q-EU`).

### 5.3 GNSS (positioning)
- **U20 — Quectel LC760Z** (LC79H-class footprint), multi-constellation receiver:
  **GPS, GLONASS, BeiDou, Galileo, QZSS**. UART to the ESP32; supplied from `VCC_LC760Z`.
- **Antenna:** external **IPEX/U.FL** connector **J3** with an active-antenna bias feed;
  RF matching and ESD on the front end.

### 5.4 Vehicle bus — CAN
- **U26 — NXP TJA1051T/3** high-speed CAN transceiver (SOIC-8, separate VIO, silent mode).
- Bus runs at **500 kbit/s** (`CAN_500_H` / `CAN_500_L` on the harness).
- Standby control via `CAN_SHDN` (S-pin: **LOW = normal**, HIGH = standby); the
  transceiver sits on the switched 3.3 V peripheral rail.
- ESP32 TWAI pins: **TX = IO40**, **RX = IO39**, **SHDN = IO41**. Firmware auto-recovers
  from bus-off.

### 5.5 Vehicle bus — RS-485 / serial
- **U22 — Maxim MAX13487EESA+T** half-duplex RS-485 transceiver with auto-direction (
  no separate DE/RE line). Signals `RS485_TX_5V` / `RS485_RX_5V`.
- **U11 — TI TXB0108RGY** 8-bit auto-direction level translator (1.2–3.6 V ↔ 1.65–5.5 V)
  bridges the 3.3 V ESP32 domain to the 5 V bus side. (`U2 TXB0102` 2-bit shifter is a
  DNP option on A42C.)
- Harness pins **10/11 (`D+/A`, `D-/B`) are dual-purpose** — USB data **or** RS-485 A/B —
  selected by population/jumpers.

### 5.6 USB
- Micro-USB (**J4**) for programming/console, with **U1 — ON NUP4202W1** USB ESD
  protection array. `USB_VBUS` sensed by the MCU.

### 5.7 Motion / inertial sensing
- **U13 — ST LIS2DH12TR** 3-axis MEMS accelerometer (±2–16 g, I²C/SPI, 12-LGA). Used for
  motion detection, crash/tip detection, tamper and low-power wake.

### 5.8 Digital inputs, ignition & tamper (opto-isolated)
- **U5, U19 — LTV-217** phototransistor optocouplers isolate vehicle-side digital inputs.
- **Ignition sense** `IGN` (harness) → opto → `IGN_IN` (**IO35**).
- **Lid / tamper:** switch `S1` → `LID_OPEN_DET` (**IO3**, boot-strap-safe); tamper loop
  `TAMPER_CONN` → `MCU_TAMPER_CONN` (**IO8**). Internal pull-ups enabled in firmware.

### 5.9 Control outputs (immobiliser / flasher)
- MCU-sourced logic lines **`T6_SIG` (IO36)** and **`IM_OUT_3V3` (IO0)** drive the
  vehicle-side power outputs **`T6_OUT`**, **`IM_OUT`** (immobiliser) and **`Flasher_Out`**
  on the harness, through bias-resistor NPN transistors (`Q4–Q6` KRC106S) and, for the
  heavy loads, the external **`flasher_relay`** companion board.
- **Note:** `IM_OUT_3V3` shares IO0 with the boot strap and boot button SW2 — driven only
  post-boot.

### 5.10 Status indication
- Three LEDs driven by the MCU on the switched rail: **STATUS** (IO48), **CONN**
  (connectivity, IO34), **NRF** (network/registration, IO37).

### 5.11 Housekeeping ICs
- **STWD100** watchdog, **CD4060** oscillator/counter, **MIC5317 / MCP1826** LDOs and
  **2N7002 / AO4407A** MOSFETs appear in the component libraries for supervisory,
  timing and switching functions.

---

## 6. Power architecture

| Rail | Source | Purpose |
|---|---|---|
| **VIN / VBAT** | Vehicle `V_EXT` (protected input) | Raw board supply |
| **5V0** | **U3 TPS566242DRLR** — 6 A buck | Main 5 V bus, RS-485, boost input |
| **4V0** | **U4 EG1192L** — 3 A buck | Cellular modem supply |
| **4V7_BOOST** | **U18 TPS61022** — boost *(XL/Plus)* | Modem PA burst support / low-batt hold-up |
| **3V3_ALWAYS_ON** | **LP5907-3.3** LDO | MCU + always-on domain |
| **3V3_LDO (switched)** | **LP5907-3.3** LDO, gated by `3V3_LDO_EN` (IO10) | CAN xcvr, sense front-ends, LEDs — power-gated when parked |
| **3V3_ANT** | **LP5907-3.3** LDO | Active-antenna bias |

**Input protection & conditioning:** `F1` resettable polyfuse; **CR1 SMCJ100A** 100 V
TVS and **CR2 PLZ15C-G3/H** zener for load-dump/surge; **Q9 AO4407A** P-channel MOSFET
for reverse-polarity protection; **U9 SQS160ELNW** MOSFET as the main load switch; bulk
electrolytics (22 µF/100 V, 470 µF/16 V) and tantalum polymer caps (100 µF) for hold-up.

**Backup battery & monitoring:** **TP4056** single-cell Li-ion charger maintains a
backup cell so the unit reports even when main power is cut (anti-theft). Battery state
is read as **IBL** ("internal battery level") — a 33k/10k divider (ratio ≈ 0.767) into
**IO2 (ADC1_CH1)**, sampled while Wi-Fi is active.

---

## 7. External interfaces

### 7.1 Main harness — J8 (18-pin, 2.5 mm right-angle, `MX23A18NF1`)

| Pin | Net | Function |
|---|---|---|
| 1 | `V_EXT` | Vehicle battery / main power in |
| 2 | `CAN_500_H` | CAN high (500 kbit/s) |
| 3 | `CAN_500_L` | CAN low |
| 4 | `Flasher_Out` | Flasher / indicator drive |
| 5 | `T6_OUT` | Aux control output (T6) |
| 6 | `IM_OUT` | Immobiliser output |
| 7 | `IGN` | Ignition sense in (opto-isolated) |
| 8 | — | (spare / no-connect) |
| 9 | `TAMPER_CONN` | Tamper loop |
| 10 | `D+ / A` | USB D+ **or** RS-485 A (dual-purpose) |
| 11 | `D- / B` | USB D- **or** RS-485 B (dual-purpose) |
| 12–17 | — | (spare / no-connect) |
| 18 | `GND` | Ground / return |

### 7.2 Antennas & connectors
- **J6** — CA-L01 LTE cellular antenna (SMD).
- **J3** — IPEX/U.FL GNSS antenna.
- **J4** — Micro-USB (console / firmware).
- **J10** — 4-pin header (aux / programming).
- Edge **Lite** uses an **11-pin JST XH** harness (`J7`) instead of the 18-pin MX header.

---

## 8. Firmware & bring-up

A board bring-up sketch — `firmware_tests/roam_edge_xl_test/roam_edge_xl_test.ino` —
targets the ESP32-S3 (Arduino-ESP32 v2/v3) and exercises four subsystems, selectable
from the **USB serial monitor (115200 baud)** *or* by a command frame on the **CAN bus**:

1. **CAN / TWAI** — TX heartbeat + verbose RX, error counters, bus-off auto-recovery,
   selectable bitrate (125/250/500/1000 kbit/s) and normal vs. self-test mode.
2. **Digital sense** — live read of ignition / lid / tamper inputs plus IBL battery
   voltage every 3 s.
3. **Status LEDs** — identify/blink STATUS, CONN, NRF.
4. **MCU outputs** — walk/drive `T6_SIG` and `IM_OUT_3V3`.

CAN command protocol: a frame to ID **0x010** with `data[0] = 0x01–0x04` selects a test
(0x00 stops); the board acks on ID **0x011**; heartbeats go out on **0x100**. The switched
peripheral rail (`3V3_LDO_EN`) is asserted at boot before any transceiver, sense line or
LED is touched.

> **Firmware pin map is authoritative against the current schematic.** The sketch header
> notes that `production/netlist.ipc` in the repo is **stale** (older net names) and must
> not be used for pin assignments.

**Confirmed ESP32-S3 pin assignments:**

| Signal | Net | GPIO | Notes |
|---|---|---|---|
| CAN TX | `CAN_TXD` | IO40 | |
| CAN RX | `CAN_RXD` | IO39 | drives HIGH at reset |
| CAN standby | `CAN_SHDN` | IO41 | LOW = normal |
| Peripheral rail EN | `3V3_LDO_EN` | IO10 | active-HIGH |
| Ignition sense | `IGN_IN` | IO35 | via LTV-217 opto |
| Lid/tamper switch | `LID_OPEN_DET` | IO3 | strap pin |
| Tamper loop | `MCU_TAMPER_CONN` | IO8 | |
| Battery level | `IBL` | IO2 | ADC1_CH1, ÷0.767 |
| Status LED | `STATUS_LED_3V3` | IO48 | |
| Connectivity LED | `CONN_LED` | IO34 | |
| Network LED | `NRF_LED` | IO37 | |
| Control out | `T6_SIG` | IO36 | |
| Control out | `IM_OUT_3V3` | IO0 | shared w/ boot strap + SW2 |

---

## 9. Key bill-of-materials (Edge XL / A42C)

| Ref | Part | Function |
|---|---|---|
| U25 | ESP32-S3-MINI-1-N8 | Application MCU (Wi-Fi/BLE, TWAI) |
| U7 | Quectel EG915U-EU | LTE Cat 1 modem |
| U20 | Quectel LC760Z | Multi-GNSS receiver |
| U26 | NXP TJA1051T/3 | CAN transceiver |
| U22 | MAX13487EESA+T | RS-485 transceiver (auto-dir) |
| U11 | TI TXB0108RGY | 8-bit level translator |
| U13 | ST LIS2DH12TR | 3-axis accelerometer |
| U6 | Winbond W25Q128JVP | 16 MiB QSPI flash |
| U1 | NUP4202W1 | USB ESD protection |
| U5, U19 | LTV-217 | Opto-isolators (ignition/inputs) |
| U3 | TI TPS566242DRLR | 6 A buck → 5V0 |
| U4 | EG1192L | 3 A buck → 4V0 (modem) |
| U18 | TI TPS61022 | Boost → 4V7 |
| U10/U21/U27 | LP5907MFX-3.3 | 3.3 V LDOs |
| U9 | SQS160ELNW-T1 | Load-switch MOSFET |
| Q9 | AO4407A | Reverse-polarity P-FET |
| — | TP4056 | Backup Li-ion charger |
| CR1 | SMCJ100A | 100 V TVS (input surge) |
| CR2 | PLZ15C-G3/H | Zener protection |
| F1 | Polyfuse (1206) | Resettable input fuse |
| J8 | MX23A18NF1 | 18-pin vehicle harness |
| J6 / J3 | CA-L01 / IPEX | Cellular / GNSS antenna |

Full per-part BOMs, including LCSC/MPN part numbers and DNP flags, live in
`EDGE_A42C_BOM_DNP.csv` (XL / A42C build) and `LITE_E4C_BOM` (Lite build).

---

## 10. Design status & open items

- **Layout:** complete and routed on 6 layers; **0 unconnected pads**, **0 footprint
  errors** (per `DRC.rpt`, 2026-05-25).
- **DRC:** **122 open violations** remain — predominantly *negative annular width* on a
  handful of PTH pads, *keepout-area* track incursions (e.g. `SIM_VDD` on BOTTOM near an
  RF keepout), and *solder-mask min-width* slivers. These should be cleared before release
  fabrication; none are connectivity errors.
- **Stale artifact:** `production/netlist.ipc` uses **old net names** — do not use for pin
  mapping (the firmware header is the source of truth).
- **Custom libraries:** vendor parts (EG915, LC760Z/LC79H, SIM8060, CA-L01) are managed in
  `kicad_libraries/` with an index (`_INDEX.md`) tracking which are done vs. "need vendor
  download". Project-specific footprints live in the `EDGE:` library.

---

## 11. Companion board — `flasher_relay`

A separate, small KiCad project (`flasher_relay/`) implements the high-current
relay/flasher stage that the Edge logic outputs (`T6_OUT`, `IM_OUT`, `Flasher_Out`)
switch. Keeping it on its own board isolates inductive relay loads and switching noise
from the sensitive RF and MCU sections of the main Edge unit. It has its own schematic,
PCB and backup history and is versioned alongside the Edge design.

---

*Specification derived from the `roam_edge_xl` KiCad project: schematic, PCB, project
settings, BOM/DNP files, position files, DRC report and the bring-up firmware. Rail
currents and modem/GNSS capabilities are per component datasheet ratings; verify against
the final populated build.*
