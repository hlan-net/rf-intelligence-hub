# Device Specification

Normative specification of the device itself — development platform, hardware,
power, security hardware, PCB path, and the device-level open decisions and
constraints. Governed by [`design_principles.md`](design_principles.md); security
detail in [`security.md`](security.md).

> Project-level context (overview, AI, companion app, cloud backend) lives in
> [`../docs/requirements.md`](../docs/requirements.md).

---

## 1. Development platform

| Component | Detail |
|---|---|
| SBC | Raspberry Pi 5 |
| AI accelerator | Hailo-10H via Raspberry Pi AI HAT+ 2 |
| Dedicated memory | 8GB LPDDR4X on-module |
| Performance | 40 TOPS INT4 |
| Purpose | Benchmarking, firmware development, model compilation — not embedded in end device |
| LLM runtime | hailo-ollama, llama.cpp |
| Target validation | Target-model HEF compilation feasibility, token/s benchmarks, current draw profiling |

---

## 2. End device hardware

### 2.1 Form factor
- Flipper Zero-inspired — handheld, pocketable
- Custom PCB designed in KiCad
- Enclosure: AnkerMake FDM for prototyping, PCBway SLS resin for final production
- Enclosure modelled parametrically in OpenSCAD with AI assistance
- KiCad 3D export used as reference geometry in OpenSCAD

### 2.2 Compute (TBD — pending dev platform benchmarks)

| Candidate | Notes |
|---|---|
| Pi Compute Module 4 | Linux-native, eMMC support, proven |
| Pi Zero 2W | Smaller, lower power, no eMMC native |
| Other SoC | i.MX 8, RK3588 if benchmarks indicate need |

Decision criteria from dev platform: sustained token/s at acceptable power draw, Hailo HEF compilation success for target models, thermal behaviour in enclosure.

### 2.3 RF subsystems

| Protocol | Module candidate | Notes |
|---|---|---|
| BLE 5 | nRF52840 or u-blox NINA-B3 | Passive advertisement scanning |
| WiFi 6 + 802.15.4 (Zigbee/Thread) + BLE | ESP32-C6 | 2.4 GHz only — single chip simplifies RF layout |

**Correction / open scope:** the ESP32-C6 has **no sub-GHz radio** — it is 2.4 GHz
only. The earlier "sub-GHz" attribution was an error. Sub-GHz (433/868 MHz)
reception would require a separate front-end (e.g. CC1101) and carries elevated
legal exposure — see [`../docs/legal_privacy.md`](../docs/legal_privacy.md) §5 and
principle P2. **Out of scope for v1** unless explicitly added (open decision).

RF layout considerations (designer has telecom background):
- Separate antenna keep-out zones per module
- Ground pour isolation between RF domains
- 50Ω trace routing for all RF signals
- ESP32-C6 antenna placement away from battery and display

All RF intelligence is **receive-only** (principle P2) — no active TX for scanning.

### 2.4 Display and input
- Display: Sharp LS013B7DH03 memory LCD (SPI)
- Input: 5-way navigation switch + 3 tactile buttons, GPIO direct
- Status LEDs: RGB, indicates scan active / sync / AI inference
- USB-C: power input + data (charges battery only, no direct system power)

### 2.5 Storage

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

### 2.6 Power architecture

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

### 2.7 Security hardware

**Locked requirement:** the device includes a **dedicated secure element** that
provides both:
- **Hardware-backed secret storage** — cloud credentials, device identity key, and
  TLS material are non-extractable and unrecoverable from a flash/eMMC dump.
- **A secure TRNG** — hardware true-random source for key generation, TLS, nonces,
  BLE pairing, and the GDPR identifier salt.

This is SoC-independent and does not wait on dev-platform benchmarks. Long-term
keys are generated inside the secure element. Desired (not required): randomness
and key material not eavesdroppable (on-die TRNG for host randomness and/or
encrypted SE I/O channel; prefer a CC-certified part).

Specific part (ATECC608B vs NXP SE050 vs Infineon OPTIGA vs TPM 2.0 SLB9670)
remains open — decided once the compute-SoC class (Linux vs MCU) is known. Full
analysis and threat model: [`security.md`](security.md).

---

## 3. PCB production path

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

## 4. Open decisions (device)

| Decision | Blocked on | Target answer by |
|---|---|---|
| End device compute SoC | Hailo-10H LLM benchmark on Pi 5 | Rev 0 validation |
| Target LLM on Hailo-10H (HEF compilation) | HEF compilation attempt | Rev 0 validation |
| eMMC vs QSPI NOR | Power draw measurement on dev platform | Rev 0 validation |
| SDIO vs SPI for microSD | Compute SoC confirmed | Post SoC decision |
| Always-on current budget | Full subsystem power profiling | Rev 0 validation |
| Supercap sizing (Farads) | Peak current spike measurement | Rev 0 validation |
| Secure element part (ATECC608B / SE050 / OPTIGA / TPM 2.0) | Compute SoC class (Linux vs MCU) | Post SoC decision |
| Sub-GHz (433/868 MHz) in scope? | Legal scope + value assessment | Pre-schematic |

*Note:* presence of a dedicated secure element (storage + TRNG) is **locked** — see
§2.7; only the specific part is open.

---

## 5. Device constraints

- **Passive RX only** — no active TX for scanning (principle P2)
- **RF keep-out zones mandatory** — antenna isolation for BLE / Zigbee / WiFi
- **USB-C charges LiPo only** — never directly powers system (RF noise isolation), except AA-sled mode (§2.6)
- **AA primary sled disables charging** — BMS firmware enforced, safety critical
- **Read-only rootfs** — overlay pattern only, factory reset = wipe overlay
- **exFAT on microSD** — field readable on any PC or Pi without tools
- **Secrets in hardware** — secure element holds credentials and provides the TRNG (§2.7)
