# GDPR — Design Notes & Data-Handling Decisions

> **Not legal advice.** Engineering-facing analysis to drive privacy-by-design.
> See [`legal_privacy.md`](legal_privacy.md) for the broader legal context.

## What counts as personal data here

| Field captured | Personal data? | Notes |
|---|---|---|
| WiFi MAC | Yes | Even randomized MACs when linkable across time/space (EDPB; CJEU *Breyer*) |
| BLE MAC / address | Yes | Same reasoning; resolvable private addresses especially |
| WiFi probe-request SSID | Yes — highest sensitivity | Reveals networks the device previously joined → home/work/travel history |
| Device name / GAP name | Often | Frequently contains a person's name or model |
| Signal strength (RSSI) | Indirect | Enables presence/proximity inference when tied to an identifier |
| Location (GPS) | Yes | Especially combined with identifier + timestamp = movement profile |
| Timestamp | Indirect | Profiling enabler in combination |

**Identifier + location + timestamp together = a tracking record.** The whole
risk concentrates where these three meet.

## Privacy-by-design levers (decide in planning, they shape firmware & schema)

| Lever | Options | Effect |
|---|---|---|
| Identifier storage | raw MAC ↔ hashed+salted ↔ counter/aggregate only | biggest GDPR-exposure reducer |
| Probe-request SSIDs | store ↔ drop entirely ↔ "probe occurred" counter only | removes the worst field |
| Location | precise GPS ↔ coarsened (e.g. 100 m) ↔ none | blocks profiling |
| Retention | indefinite ↔ auto-delete after N days | data minimisation |
| Payload | dropped at radio layer ↔ dropped at log layer | confidentiality of comms (legal constraint, not a choice) |

## Recommended defaults (for discussion)

- Hash + per-device rotating salt on identifiers at capture time; never persist raw MAC by default.
- Do **not** store probe-request SSID strings by default; keep only an event count.
- Coarsen location by default; precise GPS opt-in only.
- Default retention window with auto-expiry on device and in Firestore.
- These defaults define the BLE GATT data model (requirements §6.1) and the Firestore document schema (requirements §7.1).

## Open decisions (mirrored in [`legal_privacy.md`](legal_privacy.md))

1. Household exemption vs GDPR-compliant by design *(recommended: compliant by design)*.
2. Default identifier handling — raw vs hashed+salted.
3. Store probe-request SSIDs at all?
4. Retention window length and where enforced (device, cloud, both).
5. Separate "compliant mode" vs "research mode" firmware profiles?

Each resolved decision should update the BLE GATT profile and Firestore schema in
[`requirements.md`](requirements.md).
