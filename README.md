# RF Intelligence Hub

An open source, field-deployable RF intelligence device in a Flipper Zero-inspired
form factor. Passive RF scanning and data logging across BLE, Zigbee/802.15.4 and
WiFi, with a local + cloud AI overlay. Fully reproducible from published hardware
designs, firmware, the companion app, and documentation.

> **Status:** Design / planning phase. No hardware has been built and no firmware
> written yet — see [`ROADMAP.md`](ROADMAP.md) for where we are and what is next.

## Repository layout

| Path | Contents |
|---|---|
| [`spec/`](spec/) | **Normative specifications** — device spec, design principles, security architecture |
| [`docs/`](docs/) | Background & analysis — project overview, legal/privacy, GDPR notes |
| [`hardware/`](hardware/) | Physical device: KiCad schematics & PCB, OpenSCAD enclosure, BOM |
| [`firmware/`](firmware/) | On-device software: main compute SoC and RF scanning MCU |
| [`app/`](app/) | Flutter companion app (Android primary, iOS compatible) |
| [`cloud/`](cloud/) | Firebase backend — Cloud Functions, Firestore/GCS config |
| [`models/`](models/) | LLM / AI model artifacts (compiled HEF, GGUF, etc.) |

## Key documents

**Specifications (normative — [`spec/`](spec/)):**
- [Design principles](spec/design_principles.md) — the project-wide rules every decision must respect
- [Device specification](spec/device.md) — hardware, RF, power, security hardware, PCB path
- [Security architecture](spec/security.md) — threat model, secret storage, secure randomness, OTA signing
- [Agent / autonomy model](spec/agent.md) — purpose & state ownership, goal intake, resource-bounded planning, cloud-spend governance

**Background ([`docs/`](docs/)) & roadmap:**
- [Roadmap](ROADMAP.md) — phases and current status
- [Project requirements (overview)](docs/requirements.md) — overview, AI, app, cloud
- [Legal & privacy overview](docs/legal_privacy.md) — the three legal regimes that apply
- [GDPR design notes](docs/gdpr.md) — detailed data-protection analysis and decisions
- [Flipper One comparison](docs/flipper_one.md) — how this project compares to Flipper One
- [GNSS & time synchronization](docs/gnss.md) — offline RTC time sync and privacy notes



## Design philosophy

See [design principles](spec/design_principles.md) for the full, ratified set.

- **The device is the center, and self-sufficient** — core functions (scan, log, local inference, UI) work with no app, no cloud, no network. Everything else serves the device.
- **Dev platform first** — SoC and accelerator decisions deferred until benchmarked on Raspberry Pi 5 + AI HAT+ 2 (Hailo-10H).
- **Field-first engineering** — battery life, cold-weather operation, replaceability over raw performance.
- **Decentralized, user-owned cloud** — no central backend; each device uses the user's own cloud, provisioned per-device and authorized by the user. Cloud stays an enhancement, never a precondition.
- **Privacy by design** — passive metadata only, no communication content; see [GDPR notes](docs/gdpr.md).
- **Open and reproducible** — all hardware, firmware, app, and backend config under Apache 2.0.

## License

[Apache 2.0](LICENSE). This repository publishes designs and source only. No
assembled devices are sold or distributed; anyone building the device is the
"manufacturer" and responsible for compliance in their jurisdiction. See
[legal & privacy](docs/legal_privacy.md).
