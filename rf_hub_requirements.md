# RF Intelligence Hub — Project Requirements

**Version:** 0.1 (requirements gathering)  
**License:** Apache 2.0  
**Repository:** GitHub (to be created)  
**Author:** Lead Architect, Cloud Solutions  
**Status:** Pre-schematic, pre-prototype

---

## 1. Project overview

An open source, field-deployable RF intelligence device in a Flipper Zero-inspired form factor. Primary function is passive RF scanning and data logging across BLE, Zigbee, and WiFi protocols, with an AI agent overlay providing local inference for UI interactions and cloud-offloaded inference for heavy analysis tasks. Designed to be fully reproducible by others from published hardware designs, firmware, and documentation.

---

## 2. Design philosophy

- **Dev platform first** — all SoC and accelerator decisions are deferred until benchmarks on the Raspberry Pi 5 + AI HAT+ 2 (Hailo-10H) development platform confirm viable candidates
- **Field-first engineering** — battery life, cold weather operation, and physical replaceability take priority over raw performance
- **Cloud-light device** — the device holds no cloud credentials; the Android companion app owns all cloud interaction
- **Open and reproducible** — KiCad schematics, OpenSCAD enclosure models, firmware, Flutter app, and Firebase configuration all published under Apache 2.0

---

## 3. Development platform

| Component | Detail |
|---|---|
| SBC | Raspberry Pi 5 |
| AI accelerator | Hailo-10H via Raspberry Pi AI HAT+ 2 |
| Dedicated memory | 8GB LPDDR4X on-module |
| Performance | 40 TOPS INT4 |
| Purpose | Benchmarking, firmware development, model compilation — not embedded in end device |
| LLM runtime | hailo-ollama, llama.cpp |
| Target validation | Gemma 4 12B HEF compilation feasibility, token/s benchmarks, current draw profiling |

---

## 4. End device hardware

### 4.1 Form factor
- Flipper Zero-inspired — handheld, pocketable
- Custom PCB designed in KiCad
- Enclosure: AnkerMake FDM for prototyping, PCBway SLS resin for final production
- Enclosure modelled parametrically in OpenSCAD with AI assistance
- KiCad 3D export used as reference geometry in OpenSCAD

### 4.2 Compute (TBD — pending dev platform benchmarks)

| Candidate | Notes |
|---|---|
| Pi Compute Module 4 | Linux-native, eMMC support, proven |
| Pi Zero 2W | Smaller, lower power, no eMMC native |
| Other SoC | i.MX 8, RK3588 if benchmarks indicate need |

Decision criteria from dev platform: sustained token/s at acceptable power draw, Hailo HEF compilation success for target models, thermal behaviour in enclosure.

### 4.3 RF subsystems

| Protocol | Module candidate | Notes |
|---|---|---|
| BLE 5 | nRF52840 or u-blox NINA-B3 | Passive advertisement scanning |
| WiFi + Zigbee + sub-GHz | ESP32-C6 | Single chip covers all three, simplifies RF layout |

RF layout considerations (designer has telecom background):
- Separate antenna keep-out zones per module
- Ground pour isolation between RF domains
- 50Ω trace routing for all RF signals
- ESP32-C6 antenna placement away from battery and display

### 4.4 Display and input
- Display: Sharp LS013B7DH03 memory LCD (SPI)
- Input: 5-way navigation switch + 3 tactile buttons, GPIO direct
- Status LEDs: RGB, indicates scan active / sync / AI inference
- USB-C: power input + data (charges battery only, no direct system power)

### 4.5 Storage

| Layer | Type | Capacity | Filesystem | Purpose |
|---|---|---|---|---|
| Onboard flash | eMMC (soldered) | ~4GB | — | OS, firmware, AI model cache |
| Field storage | microSD (push-push connector) | User-supplied | exFAT | RF scan data, logs, AI reports |

**eMMC partition layout:**
- Boot: ~64MB FAT32
- Root filesystem: ~2GB ext4 read-only squashfs
- Overlay: ~256MB ext4 writable (config, calibration, credentials)
- AI model cache: ~1GB ext4

**microSD data layout:**
```
/rf_scans/       — RF capture files
/ble_logs/       — BLE advertisement logs  
/wifi_probes/    — WiFi probe data
/ai_reports/     — Generated reports
```

**OS pattern:** Hybrid — read-only rootfs + writable overlay partition for config only. Factory reset = wipe overlay. OTA = atomic rootfs image replacement, overlay survives.

**eMMC power draw:** To be validated on dev platform. If idle draw exceeds budget, evaluate QSPI NOR flash as fallback.

**Interface mode:** SDIO vs SPI to be decided after compute SoC is confirmed.

### 4.6 Power architecture

**Primary source — normal operation:**
- Single 18650 Li-ion cell in swappable sled
- Recommended cells: Samsung 30Q or Panasonic NCR18650B
- BMS IC: BQ25895 configured for 4.2V Li-ion charge profile
- USB-C slow charge: 5W (5V/1A), no fast charge, no PD negotiation required

**Primary source — cold weather operation (Finland, to -25°C):**
- 2× Energizer Ultimate Lithium AA in swappable sled
- Primary cells — not rechargeable
- BMS charging disabled when AA sled detected
- USB-C can power system directly when AA sled installed, will not charge

**Swappable sled system:**
- Single physical bay: 30×24×105mm internal cavity
- Two 3D-printed sleds with identical outer dimensions and connector position
- 3-pin connector: VBAT+, GND, ID
- ID resistor in sled: 0Ω = 18650 Li-ion, 100kΩ = 2×AA primary
- BMS firmware reads ID on boot via ADC, configures charge profile accordingly
- Lanyard hole on sled for cold-weather field use with gloves
- NTC thermistor against cell: BQ25895 refuses charge below 0°C

**Burst buffer:**
- Supercapacitor buffer on output side of boost converter
- Charger IC: LTC3225 (current-limited trickle from LiPo)
- Handles display, WiFi TX, AI inference current spikes without LiPo voltage droop

**Boost converter:**
- IC: TPS61023
- Input: 0.5–4.2V (covers both 18650 and deeply discharged 2×AA at -25°C)
- Output: regulated 3.8V system rail

**Power domains:**

| Domain | Source | Components | Notes |
|---|---|---|---|
| Always-on | LiPo direct, 3.3V LDO (TPS62840) | ESP32-C6, RTC, BLE passive, logging MCU | 60nA quiescent, runs continuously |
| Burst | Supercap buffered, switchable | Compute SoC, display, WiFi TX, active probing | Spun up on user interaction |
| USB | When cable present | Charges LiPo only | Never directly powers system |

Target always-on current: <100mA sustained. To be validated on dev platform.

---

## 5. AI architecture

### 5.1 Local inference (on device)
- Runtime: llama.cpp or hailo-ollama (pending benchmark)
- Target model: Gemma 3 1B (confirmed fits) or Gemma 4 12B via Hailo HEF (to be validated)
- Use cases: UI interactions, quick natural language queries, on-screen summaries
- Constraint: must not drain burst power domain; inference triggered on user request only

### 5.2 Cloud inference (via companion app)
- Provider: Google Gemini API (Vertex AI)
- Routing: Flutter app proxies queries from device via BLE → Firebase Cloud Functions → Gemini
- Use cases: deep RF analysis, anomaly correlation, session report generation, NL queries over historical data
- Model selection: Gemini Flash for high-volume/fast tasks, Gemini Pro for deep analysis

### 5.3 Hybrid routing logic
```
User query on device
    → local Gemma handles if: simple status / quick summary / no internet
    → escalates to cloud if: complex analysis / large context / multimodal
    → cloud result returned to device via BLE from app
```

---

## 6. Companion Android app (Flutter)

**Technology:** Flutter (Dart) — Android primary, iOS compatible from same codebase  
**Communication:** BLE GATT only — no WiFi required for device-app link  
**Backend:** Firebase (Firestore, GCS, Cloud Functions, Firebase Auth)

### 6.1 BLE GATT profiles

| Profile | Direction | Purpose |
|---|---|---|
| OTA | App → Device | Chunked firmware delivery, verify, apply |
| Scan control | App → Device | Start/stop/configure RF sessions |
| Live data stream | Device → App | Real-time RF observations notify |
| AI query | Bidirectional | Send question, stream response back |
| Config | Bidirectional | Device settings, thresholds, parameters |
| Status/telemetry | Device → App | Battery, temp, storage, uptime |
| Export trigger | App → Device | Initiate bulk data package for cloud sync |

### 6.2 OTA firmware update flow
```
GCS bucket (new firmware image)
    → Cloud Function validates + signs
    → Flutter app downloads on WiFi
    → App connects to device via BLE
    → Chunked transfer via OTA GATT profile
    → Device verifies hash, writes to inactive rootfs partition
    → Device reboots into new rootfs, overlay survives
```

### 6.3 Cloud sync flow
```
User triggers sync in app
    → App connects to device via BLE
    → Export trigger GATT profile initiates data package
    → Device streams enriched scan records over BLE
    → App batches and writes to Firestore
    → Raw files uploaded to GCS
```

---

## 7. Cloud backend (Google / Firebase)

**Provider:** Google Cloud + Firebase  
**Architecture:** Serverless — no VMs, no Kubernetes, no self-hosted inference

| Service | Purpose |
|---|---|
| Firebase Auth | User authentication (personal use, single user initially) |
| Firestore | Scan session documents, enriched with AI labels |
| Google Cloud Storage | Raw RF capture files, firmware images |
| Cloud Functions (Python) | Gemini API proxy, OTA signing, data enrichment triggers |
| Vertex AI / Gemini API | Heavy AI inference, report generation |

### 7.1 Firestore document structure (per scan session)
```json
{
  "session_id": "uuid",
  "timestamp_start": "ISO8601",
  "timestamp_end": "ISO8601",
  "location": { "lat": 0.0, "lng": 0.0 },
  "protocols_scanned": ["BLE", "WiFi", "Zigbee"],
  "observations": [...],
  "ai_labels": [...],
  "anomaly_flags": [...],
  "ai_summary": "Gemini-generated narrative",
  "raw_file_gcs": "gs://bucket/path/to/capture.bin",
  "device_id": "uuid",
  "firmware_version": "semver"
}
```

---

## 8. PCB production path

| Phase | Activity | Tooling |
|---|---|---|
| Rev 0 | Breakout board breadboard validation | Dev platform + off-shelf modules |
| Schematic | KiCad schematic, component selection | KiCad + PCBway component library |
| Layout | PCB layout, RF keep-out zones, 50Ω traces | KiCad |
| Rev 1 | 5-board prototype run, PCBA | PCBway PCBA + KiCad plugin |
| Enclosure v1 | FDM prototype | AnkerMake + OpenSCAD |
| Validation | RF performance, power draw, thermals | Dev platform instruments |
| Rev 2 | Final PCBA run | PCBway PCBA |
| Enclosure final | SLS/resin production quality | PCBway 3D print |

---

## 9. Open source repository structure

```
rf-intelligence-hub/
├── hardware/
│   ├── kicad/          — schematic + PCB layout
│   ├── openscad/       — parametric enclosure models
│   │   ├── main_body.scad
│   │   ├── sled_18650.scad
│   │   └── sled_2xAA.scad
│   └── bom/            — PCBway BOM files
├── firmware/
│   ├── device/         — main compute SoC firmware (Python/C)
│   ├── rf_mcu/         — ESP32-C6 RF scanning firmware
│   └── models/         — Hailo HEF compiled model files
├── app/
│   └── flutter/        — companion Android/iOS app
├── cloud/
│   └── functions/      — Firebase Cloud Functions (Python)
├── docs/
│   ├── build_guide.md
│   ├── bom_sourcing.md
│   ├── pcbway_ordering.md
│   └── firebase_setup.md
└── LICENSE             — Apache 2.0
```

---

## 10. Open decisions (pending dev platform benchmarks)

| Decision | Blocked on | Target answer by |
|---|---|---|
| End device compute SoC | Hailo-10H LLM benchmark on Pi 5 | Rev 0 validation |
| Gemma 4 12B on Hailo-10H | HEF compilation attempt | Rev 0 validation |
| eMMC vs QSPI NOR | Power draw measurement on dev platform | Rev 0 validation |
| SDIO vs SPI for microSD | Compute SoC confirmed | Post SoC decision |
| Always-on current budget | Full subsystem power profiling | Rev 0 validation |
| Supercap sizing (Farads) | Peak current spike measurement | Rev 0 validation |

---

## 11. Key constraints summary

- **No cloud credentials on device** — app owns all cloud auth and sync
- **No Kubernetes or self-hosted inference** — serverless only on backend
- **RF keep-out zones mandatory** — antenna isolation for BLE / Zigbee / WiFi
- **USB-C charges LiPo only** — never directly powers system (RF noise isolation)
- **AA primary sled disables charging** — BMS firmware enforced, safety critical
- **Read-only rootfs** — overlay pattern only, factory reset = wipe overlay
- **exFAT on microSD** — field readable on any PC or Pi without tools
- **Apache 2.0** — all hardware, firmware, app, and cloud config

---

*Requirements gathered through iterative design sessions. Next step: Rev 0 breadboard validation on Pi 5 + AI HAT+ 2 dev platform.*
