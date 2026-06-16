# Security Architecture

Consolidates the device's security design. Driven by the decentralized,
user-owned cloud model (see [`design_principles.md`](design_principles.md) P3),
which puts user-supplied cloud credentials **on the device**.

## Threat model (primary)

**Device theft / physical access.** The attacker can take the device apart,
desolder the eMMC / flash, and dump it. Therefore:

- Secrets must **not** be recoverable from a flash/eMMC dump.
- Software-only encryption on readable flash is insufficient (the key would have
  to live on the same readable medium). A **hardware root of trust** is required.

Secondary: BLE eavesdropping/MITM during provisioning; malicious/tampered
firmware; weak/predictable randomness undermining key generation; supply-chain
(out of scope for v1 beyond signed releases).

## Secure secret storage (requirement)

Secrets to protect: cloud access/refresh tokens (P3), device identity key
(attestation / OTA), TLS client material.

**Requirement:** hardware-backed, non-extractable root key; secrets unrecoverable
after theft; cryptographically erasable on factory reset.

> **Locked (2026-06-16):** the device *will* include a dedicated secure element
> providing both secret storage and a TRNG — see [`device.md`](device.md)
> (Security hardware). Only the specific part is still open.

### Approach

1. **Dedicated external secure element on I²C** — keeps this decision
   *independent of the compute-SoC choice* (still TBD per P7), so it does not
   block on benchmarks. Candidates:
   | Part | Bus | Notes |
   |---|---|---|
   | Microchip ATECC608B | I²C | Cheap, ECC P-256, keys non-extractable; small data slots |
   | NXP SE050 (EdgeLock) | I²C | Larger secure-object storage, RSA+ECC, certs/attestation |
   | Infineon OPTIGA Trust M | I²C | Similar class to SE050 |
   | Infineon SLB9670 (TPM 2.0) | SPI | Best if compute SoC runs Linux — measured boot, NV storage |

2. **Wrap-key pattern for large secrets.** OAuth refresh tokens can exceed an
   SE's data-slot size. Keep a non-extractable wrapping key in the SE; store the
   secret as ciphertext in normal flash; the SE wraps/unwraps. Sidesteps storage
   limits even with a small SE like the ATECC.

3. **Defense-in-depth via ESP32-C6 built-ins.** Enable Secure Boot v2 + flash
   encryption + the eFuse-backed Digital Signature (DS) peripheral so keys never
   appear in readable flash.

4. **Cryptographic erase.** Factory reset (wipe overlay — see [`device.md`](device.md) Storage) also
   destroys the SE wrapping key → all stored secrets become unrecoverable at once.

### Architectural option — ESP32-C6 as the "secure connectivity processor"

The ESP32-C6 already owns WiFi (the direct uplink), BLE (the provisioning channel
and app link), and has Secure Boot + flash encryption + DS peripheral. It is a
natural home for credentials and the cloud TLS session, keeping the Linux compute
SoC **out of the secret path** entirely (good least-privilege). Worth evaluating
against a dedicated SE on the dev platform.

## Secure randomness (TRNG / CSPRNG) — requirement

The device must generate cryptographically secure random numbers itself, from a
**hardware entropy source**, for: key generation, TLS, nonces, BLE pairing, and
the per-device rotating **salt** used to hash captured identifiers (P4 / GDPR).
A weak or predictable RNG would undermine every other measure here.

### Sources available (synergy with the secure-element decision)

- **Secure element TRNG (preferred for key generation).** ATECC608B, SE050,
  OPTIGA, and TPM 2.0 all include a certified hardware RNG. Best pattern: generate
  long-term keys (device identity, wrapping keys) **inside the SE** using its TRNG
  — the private key never exists outside the hardware. The SE we pick for secure
  storage therefore also solves randomness.
- **ESP32-C6 hardware RNG.** Usable, but **only truly random when RF (WiFi/BT) is
  enabled or the ADC entropy source is active** — Espressif documents that early
  boot with RF off yields pseudo-random output. Firmware must condition entropy
  before relying on it, especially during provisioning/key-gen.
- **nRF52840** RNG + CC310 CryptoCell TRNG (CTR_DRBG) — good quality if a BLE MCU
  is present.
- **Linux compute SoC** (if chosen): SoC hardware RNG via `/dev/hwrng` seeds the
  kernel CSPRNG; use `getrandom()` — never roll your own.

### Rules

- Combine/condition multiple sources; **never** rely on a single source at first
  boot, and do not generate long-term keys before the entropy pool is seeded.
- Prefer in-SE key generation so private keys are born and stay in hardware.
- The GDPR identifier salt (P4) must come from this CSPRNG, not a fixed constant.

### Desired (not required): not eavesdroppable

A wish, not a hard requirement: the randomness should resist interception — both
bus probing and side-channel (power/EM) observation. Concrete levers:

- **Bus exposure.** Raw random bytes pulled from an *external* SE travel over I²C
  and could be probed. Mitigations: (a) prefer the **on-die** TRNG of the host
  SoC/MCU (ESP32-C6, nRF CC310, Linux `/dev/hwrng`) for host-consumed randomness —
  never crosses a bus; (b) if using an external SE, enable its **encrypted I/O
  channel** (ATECC608B IO-protection key, or SE050 SCP03) so bus traffic is not
  plaintext.
- **Keys never on the bus.** Generate long-term keys *inside* the SE so raw key
  material never crosses any bus regardless of randomness source.
- **Side-channel.** Prefer a Common Criteria-certified part (e.g. SE050 is CC
  EAL6+) where TRNG side-channel resistance is in scope. Treat as a tie-breaker in
  selection, not a gate.

## BLE provisioning channel

- Use **LE Secure Connections** (bonded, MITM-protected) for the credential
  handoff (P3 step 3).
- Plaintext credential exists only transiently in RAM, written straight into the
  SE / encrypted store; never logged, never persisted in the clear.

## OTA signing (no central infrastructure)

Per P3 there is no central backend, so OTA cannot rely on a central signing
function. Instead: firmware images are **signed at release time** with the
project's release key; the device verifies against a **baked-in public key**
before writing to the inactive rootfs (the [`../docs/requirements.md`](../docs/requirements.md)
§6.2 OTA flow otherwise stands).

## Open decisions

1. Secure-element class: ATECC608B (cheap, wrap-pattern) vs SE050/OPTIGA (large
   secure objects) vs TPM 2.0 (Linux compute SoC + measured boot). Depends on
   compute-SoC class and attestation needs — defer final pick, but an external SE
   keeps us unblocked.
2. Dedicated SE vs ESP32-C6-as-secure-connectivity-processor (or both).
3. Credential type & lifetime (short-lived tokens + refresh) — see P3 follow-ups.
4. Is Secure Boot mandatory for v1, or hardening for v2?
5. Where are long-term keys generated — inside the SE (preferred) vs on the host
   from a seeded CSPRNG? Confirm the entropy-conditioning rule for the ESP32-C6
   path.
6. *(Wish, not gate)* Minimize randomness interception: on-die TRNG for
   host-consumed randomness and/or encrypted SE I/O channel; prefer a CC-certified
   SE for side-channel resistance.
