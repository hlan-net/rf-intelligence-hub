# Roadmap

This project is intentionally in a **multi-week design / planning phase**. The
goal right now is to plan, not build. Nothing below is committed to silicon or
code until the open questions it depends on are resolved.

## Current phase — Design & planning

- [x] Initial requirements gathered ([`docs/requirements.md`](docs/requirements.md), [`spec/device.md`](spec/device.md))
- [x] Repository structure (normative `spec/` vs background `docs/`)
- [x] Design principles ratified ([`spec/design_principles.md`](spec/design_principles.md), P1–P7)
- [x] Cloud model locked — decentralized, user-owned (P3)
- [x] Security: secure element (storage + TRNG) locked ([`spec/security.md`](spec/security.md))
- [ ] Legal & privacy design decisions ([`docs/legal_privacy.md`](docs/legal_privacy.md), [`docs/gdpr.md`](docs/gdpr.md))
- [ ] AI architecture: lock target model(s), goal model, and local/cloud split
- [ ] Power architecture review (domains, sled system, supercap, cold-weather)
- [ ] Dev-platform benchmark plan (what to measure, pass/fail thresholds)

## Phase 0 — Dev-platform validation (Rev 0)

Benchmark on Raspberry Pi 5 + AI HAT+ 2 (Hailo-10H) to unblock the open
decisions below. Breadboard validation with off-the-shelf modules.

| Decision | Blocked on |
|---|---|
| End-device compute SoC | Hailo-10H LLM benchmark on Pi 5 |
| Target LLM feasibility (HEF compilation) | Compilation attempt |
| eMMC vs QSPI NOR | Idle power-draw measurement |
| Always-on current budget | Full subsystem power profiling |
| Supercap sizing | Peak current-spike measurement |

## Phase 1 — Schematic & first PCB (Rev 1)

- KiCad schematic, component selection, RF keep-out zones, 50 Ω routing
- 5-board prototype run (PCBway PCBA)
- FDM enclosure prototype (OpenSCAD → AnkerMake)
- RF performance, power, thermal validation

## Phase 2 — Refinement (Rev 2)

- Final PCBA run
- SLS/resin production-quality enclosure
- Firmware hardening, OTA flow, companion app feature-complete

## Open decisions

Tracked in detail in [`spec/device.md`](spec/device.md) §4 (device/hardware),
[`spec/security.md`](spec/security.md) (security), and [`docs/gdpr.md`](docs/gdpr.md)
(data handling). Resolve before the dependent phase begins.
