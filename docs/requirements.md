# RF Intelligence Hub — Project Requirements (overview)

**Version:** 0.2 (design phase)
**License:** Apache 2.0
**Status:** Pre-schematic, pre-prototype

> **Scope of this document:** project-level context — overview, AI architecture,
> companion app, and cloud backend. This is background/context material.
>
> The **normative specifications** live in [`../spec/`](../spec/):
> - device hardware/power/RF/security/PCB → [`../spec/device.md`](../spec/device.md)
> - ratified rules → [`../spec/design_principles.md`](../spec/design_principles.md)
> - security architecture → [`../spec/security.md`](../spec/security.md)
>
> Legal/privacy background: [`legal_privacy.md`](legal_privacy.md), [`gdpr.md`](gdpr.md).

---

## 1. Project overview

An open source, field-deployable RF intelligence device in a Flipper Zero-inspired form factor. Primary function is passive RF scanning and data logging across BLE, Zigbee, and WiFi protocols, with an AI agent overlay providing local inference for UI interactions and cloud-offloaded inference for heavy analysis tasks. Designed to be fully reproducible by others from published hardware designs, firmware, and documentation.

## 2. Design philosophy

The ratified, canonical principles are in
[`../spec/design_principles.md`](../spec/design_principles.md) (P1–P7). In brief:
the **device is the center and self-sufficient** (P1); **passive RX only** (P2);
**decentralized, user-owned cloud** with no central infrastructure (P3);
**privacy by design** (P4); **field-first** (P5); **open and reproducible** (P6);
**dev-platform first** (P7).

---

## 3. AI architecture

### 3.1 Local inference (on device)
- Runtime: llama.cpp or hailo-ollama (pending benchmark)
- Target model: **to reconcile** — requirements have referenced both "Gemma 3 1B" (confirmed fits) and a larger Hailo-HEF model; the local model must be capable enough to run the UI standalone (P1)
- Use cases: UI interactions, quick natural language queries, on-screen summaries
- Constraint: must not drain burst power domain; inference triggered on user request only

### 3.2 Cloud inference
- Provider: Google Gemini API (Vertex AI), in the **user's own** cloud project (P3)
- Routing: the local Gemma may call cloud resources **directly**; transport is the
  opportunistic dual-path of P3 (device-direct WiFi when available, else
  app-tunneled over BLE with TLS end-to-end). The app is not a mandatory proxy.
- Use cases: deep RF analysis, anomaly correlation, session report generation, NL queries over historical data
- Model selection: Gemini Flash for high-volume/fast tasks, Gemini Pro for deep analysis

### 3.3 Hybrid routing logic
```
User query on device
    → local Gemma handles if: simple status / quick summary / no internet
    → escalates to cloud if: complex analysis / large context / multimodal
    → cloud reached directly (WiFi) or relayed via app (BLE), per P3
```

---

## 4. Companion Android app (Flutter)

**Technology:** Flutter (Dart) — Android primary, iOS compatible from same codebase
**Device link:** BLE GATT. **Cloud:** the app provisions the user's own backend and
holds the user's cloud auth (P3); it is not a required runtime proxy for the device.

### 4.1 BLE GATT profiles

| Profile | Direction | Purpose |
|---|---|---|
| OTA | App → Device | Chunked firmware delivery, verify, apply |
| Scan control | App → Device | Start/stop/configure RF sessions |
| Live data stream | Device → App | Real-time RF observations notify |
| AI query | Bidirectional | Send question, stream response back |
| Provisioning | App → Device | Hand over scoped cloud credentials (P3) |
| Cloud relay | Bidirectional | Tunnel device cloud traffic when no direct WiFi (P3) |
| Config | Bidirectional | Device settings, thresholds, parameters |
| Status/telemetry | Device → App | Battery, temp, storage, uptime |
| Export trigger | App → Device | Initiate bulk data package for cloud sync |

### 4.2 OTA firmware update flow
```
Release pipeline signs firmware image (release key)        ← no central runtime service
    → Flutter app downloads the signed image
    → App connects to device via BLE
    → Chunked transfer via OTA GATT profile
    → Device verifies signature against baked-in public key
    → Device writes to inactive rootfs, reboots, overlay survives
```
See [`../spec/security.md`](../spec/security.md) (OTA signing).

### 4.3 Cloud sync flow
```
User triggers sync in app
    → Device writes enriched scan records to the user's own cloud
       (directly over WiFi, or relayed via the app over BLE — P3)
    → Records to Firestore, raw files to GCS, in the user's project
```

---

## 5. Cloud backend (Google / Firebase) — per-user, self-provisioned

**Model:** decentralized (P3). There is **no central backend**. Each device uses
the **user's own** Google Cloud / Firebase project, provisioned per-device by the
companion app and authorized by the user. The project publishes this as a
**deployable template**, not a hosted service. Serverless only — no VMs, no
Kubernetes, no self-hosted inference.

| Service | Purpose |
|---|---|
| Firebase Auth | User authentication |
| Firestore | Scan session documents, enriched with AI labels |
| Google Cloud Storage | Raw RF capture files |
| Cloud Functions (Python) | Data-enrichment triggers, report generation |
| Vertex AI / Gemini API | Heavy AI inference, report generation |

OTA signing is **not** a cloud function (it is release-time; see §4.2 / security).

### 5.1 Firestore document structure (per scan session)
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
> The identifier and retention fields here depend on the privacy decisions in
> [`gdpr.md`](gdpr.md) (e.g. hashed vs raw identifiers, location coarsening).

---

## 6. Repository structure

See the layout table in [`../README.md`](../README.md) (canonical).

---

## 7. Open decisions

- **Device / hardware:** see [`../spec/device.md`](../spec/device.md) §4.
- **Security:** see [`../spec/security.md`](../spec/security.md) (open decisions).
- **Privacy / data handling:** see [`gdpr.md`](gdpr.md).
- **Project-level:**
  - AI target-model reconciliation (Gemma 3 1B vs a larger Hailo-HEF model) — must satisfy P1 standalone-UI requirement.
  - Credential type & lifetime for the user-owned cloud (see design principles P3 follow-ups).

---

## 8. Key project constraints

Device-level constraints are in [`../spec/device.md`](../spec/device.md) §5. Project-level:

- **Decentralized, user-owned cloud** — no central backend; the device holds the user's scoped credentials (P3)
- **No Kubernetes or self-hosted inference** — serverless only on backend
- **Apache 2.0** — all hardware, firmware, app, and cloud config
- **Designs and source only** — no assembled devices sold; the builder is the manufacturer (RED/CE)

---

*Requirements gathered through iterative design sessions. Next step: Rev 0 breadboard validation on Pi 5 + AI HAT+ 2 dev platform.*
