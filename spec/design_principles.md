# Design Principles

The ratified, project-wide rules. Every other decision must be consistent with
these. When a requirement or feature conflicts with a principle, the principle
wins (or the principle is explicitly revised here first).

## P1 — The device is the center, and the device is self-sufficient

Everything else — the companion app, the cloud backend, the model pipeline, the
dev platform — exists to **serve the device**, never the reverse. The device is
its own thing.

- **Core functions stand alone:** scanning, logging, local inference, and the UI
  work fully with **no app, no cloud, no network**. App and cloud are
  *enhancements* (sync, heavy AI, OTA, reports), never preconditions for core
  operation.
- **The device owns its identity and data locally** — field-readable microSD
  (exFAT). It can operate indefinitely, disconnected, in the field.
- **Implication / watch-point:** never let a core path depend on the app. Local
  Gemma must handle on-device UI and quick queries; cloud is escalation only.

## P2 — Passive RX only; no active TX for scanning

RF intelligence is receive-only. **No active TX for scanning purposes** — no
replay, emulation, jamming, or probe injection. This is the line that keeps the
project clear of the regulatory category that drew Flipper Zero's bans — see
[`../docs/legal_privacy.md`](../docs/legal_privacy.md) §5.

Connectivity transmissions are a separate, permitted matter: the BLE GATT link to
the companion app, and the device's optional WiFi-station uplink to the user's own
network (P3). These carry the device's own traffic — they are not RF-intelligence
emissions.

## P3 — Decentralized, user-owned cloud (no central infrastructure)

The project runs **no central backend**. There is no shared service and no
central data honeypot. Instead:

- Each device uses the **user's own cloud**, provisioned per-device and authorized
  by the user (via the companion app). Every user owns their data and their cloud
  bill.
- The local Gemma model **may use cloud resources directly** — the app is not a
  mandatory proxy for every query.
- The device therefore **holds cloud credentials** (user-supplied). This is a
  deliberate reversal of the earlier "no credentials on device" stance and makes
  credential handling a first-class security problem:
  - encrypted credential storage (prefer a secure element),
  - **least-privilege** credentials scoped to this device's own resources only,
  - **revocable** by the user from their cloud console / the app.
- Cloud stays an *enhancement* per P1: core functions work offline; cloud is used
  only when provisioned and online.

**Supersedes** requirements §2 / §11 ("no cloud credentials on device") and
reshapes §5.2 (routing) and §7 (backend) — those need reconciling. OTA signing
(§6.2) moves from a central Cloud Function to a release-time signature verified
against a baked-in public key on the device.

### Resolved (2026-06-16)

**Internet uplink — opportunistic dual-path.** The device prefers its own WiFi
(ESP32-C6 station mode) when configured/available and falls back to an
app-tunneled path over BLE otherwise. Firmware needs a single "cloud transport"
abstraction with two interchangeable backends (direct HTTPS / BLE relay). Both use
the same credentials and TLS **end-to-end to the user's cloud**, so the app is a
dumb relay that cannot read the traffic.

**Provisioning — app provisions, device gets scoped credentials.**
1. User authenticates to their cloud (Google OAuth) in the companion app.
2. App provisions per-device resources in the user's own project from a template
   (Firestore / GCS / Functions) and mints a **least-privilege, device-scoped**
   credential.
3. App hands the credential to the device over the secure BLE channel; the device
   stores it encrypted (secure element).
4. Device uses it directly (WiFi) or via the BLE relay. The user can revoke it at
   any time from their cloud console / the app.

The device never holds privileged *provisioning* credentials, and the maintainer
is never in the data path.

### New follow-up decisions

- **Credential type & lifetime:** avoid long-lived service-account keys on device
  (theft risk). Prefer short-lived tokens with a refresh path, or Firebase custom
  tokens — needs a refresh strategy that works on *both* uplink paths.
- **ESP32-C6 scan-vs-uplink contention:** the same 2.4 GHz radio cannot reliably
  scan and serve a WiFi uplink at once. Decide a time-sharing policy (pause
  scanning during uplink bursts) or whether the uplink justifies a separate RF
  resource.

## P4 — Privacy by design

Passive metadata only — **never communication content** (legal constraint, not a
choice). Data minimization by default: hash identifiers, drop the most sensitive
fields, coarsen location, bounded retention. See [`../docs/gdpr.md`](../docs/gdpr.md).

## P5 — Field-first engineering

Battery life, cold-weather operation (to −25 °C), and physical replaceability take
priority over raw performance.

## P6 — Open and reproducible

KiCad schematics, OpenSCAD enclosure, firmware, Flutter app, and Firebase config
all published under Apache 2.0. The project publishes designs and source only —
no assembled devices are sold; the builder is the "manufacturer" (RED/CE).

## P7 — Dev-platform first

SoC and accelerator decisions are deferred until benchmarked on the Raspberry Pi 5
+ AI HAT+ 2 (Hailo-10H) dev platform. See [`../ROADMAP.md`](../ROADMAP.md).

## P8 — The device owns its purpose and state; it may outsource thinking, never ownership

The device is a goal-owning agent. It owns — locally — its **purpose**
(objectives/mission), its **state** (task list, observations, knowledge), and its
**identity** (extends P1).

- **Thinking may be outsourced, ownership may not.** Heavy reasoning / strategy
  generation can run on external resources (cloud Gemini), but the result is
  *advisory* — the device incorporates it into its own plan and remains the
  authority over its goals and actions.
- **External resources are administratively the device's own.** The device has its
  own cloud *project*; the *billing account* is the user's. Ownership split:
  **billing = user, project/resources = device.**
- **Resource-bounded behaviour.** The device balances objectives against two
  finite budgets — **energy** (battery) and **money** (cloud spend). Cloud
  escalation is the most expensive action on *both* axes (money + uplink energy)
  and is used sparingly. Full model: [`agent.md`](agent.md).
