# Firmware

On-device software.

| Path | Contents |
|---|---|
| `device/` | Main compute SoC firmware (Python/C): UI, local inference, BLE GATT, OTA, storage |
| `rf_mcu/` | RF scanning firmware for the ESP32-C6 (WiFi + 802.15.4) and BLE front-end |

**Capture-path constraint:** communication *content* (payloads) must be discarded
at the radio layer and never logged — see [`docs/legal_privacy.md`](../docs/legal_privacy.md)
§1. Identifier handling (raw vs hashed) is an open decision in
[`docs/gdpr.md`](../docs/gdpr.md).

Compiled AI model artifacts live in [`../models/`](../models/), not here.
