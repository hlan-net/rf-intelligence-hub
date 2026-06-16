# Companion App

Flutter (Dart) companion app — Android primary, iOS compatible from the same
codebase.

- **Device link:** BLE GATT only (no WiFi required between app and device).
- **Backend:** Firebase — see [`../cloud/`](../cloud/).
- **Role:** owns all cloud interaction and credentials; the device holds none.

Responsibilities: OTA delivery, scan control, live data stream, AI query proxy,
config, telemetry, and cloud sync (GATT profiles in requirements §6.1).

The Flutter project will live directly under this directory.
