# Legal & Privacy — Overview

> **Not legal advice.** This document captures design-relevant legal analysis so
> that we make the right engineering decisions early. Before public release or
> wider field use, have the conclusions reviewed by a qualified lawyer. Primary
> jurisdiction assumed: Finland / EU.

A passive RF scanner does **not** raise a single "is scanning legal" question.
Three separate legal regimes apply, and each needs its own design decision.

## 1. Confidentiality of communications

Finland: *Act on Electronic Communications Services (917/2014)*; EU background:
*ePrivacy Directive (2002/58/EC)*. The decisive line is between **header / beacon
metadata** and **communication content**:

| Likely OK (passive) | Prohibited |
|---|---|
| BLE advertisement frames | Decrypting encrypted traffic |
| WiFi beacon frames | Storing unicast data-frame payloads |
| WiFi probe requests | Intercepting peer-to-peer communication |
| 802.15.4 beacons | Any attempt to read communication content |

**Design constraint (not a usage policy):** content must be discarded at the radio
layer, never written to logs. This shapes the firmware capture path.

## 2. Data protection (GDPR)

Even a single captured identifier is **personal data**. Detailed analysis and the
data-handling decisions live in [`gdpr.md`](gdpr.md). Summary:

- MAC addresses (WiFi and BLE) are personal data — including randomized MACs when linkable.
- WiFi probe-request SSIDs are the single most sensitive field (reveal a person's prior networks → home/work/travel history).
- Identifier + location + timestamp = a movement profile.

**Do not** rely on the household exemption (see §3); design as if GDPR applies.

## 3. The household exemption — and why it likely does not save us

GDPR Art. 2(2)(c) exempts purely personal/household activity. Tempting given the
"single user, personal use" framing, but:

- CJEU *Ryneš*: once a device covers **public space**, the exemption falls away. A field RF scanner used in public collects bystanders' data.
- Publishing open-source designs does not itself break the exemption (you publish plans, not data) — but **others who build the device** are not covered by *your* household use.

Working assumption: **plan for GDPR to apply.**

## 4. CE / RED and "who is the manufacturer"

The *Radio Equipment Directive (2014/53/EU)*, CE marking, and conformity duties
fall on the **manufacturer / the party placing the device on the market**.
Because this project publishes designs and source only and sells no assembled
units, those duties pass to whoever builds the device. This is favourable but
must be stated explicitly:

- A disclaimer in [`README.md`](../README.md) and `LICENSE` context: designs only, no devices sold.
- Note that the builder assumes manufacturer responsibilities (RED, CE, local frequency rules).

## 5. Reference case: Flipper Zero

Flipper Zero is the closest precedent and a useful cautionary tale.

- **Open source did not confer legality.** Its firmware is open source (GPL, on
  GitHub); this protected it from none of the actions below.
- **Canada** announced intent to ban it (Feb 2024) over **car theft** (sub-GHz
  replay / key cloning). **Amazon (US)** delisted it as a card-skimming device;
  imports have been blocked elsewhere.
- **The trigger was active/transmit capability, not passive listening** — sub-GHz
  TX (replay/jam), 125 kHz RFID and NFC emulation, IR, BadUSB. Passive reception
  was never the problem.

**Design principles we take from this:**

> **Passive RX only for RF intelligence. No active TX for scanning** — no replay,
> emulation, jamming, or probe injection. The only permitted transmission is the
> BLE GATT link to the companion app.

> **Sub-GHz is a legal decision, not just a technical one.** ESP32-C6 has *no*
> sub-GHz radio (requirements §4.3 lists "sub-GHz" against it — an error). Adding
> a real sub-GHz front-end (e.g. CC1101) to listen on 433/868 MHz imports exactly
> the capability class that drew Flipper's bans, even if receive-only. Decide
> deliberately whether sub-GHz is in scope at all.

## Open decisions

1. Rely on household exemption, or design GDPR-compliant from the start? *(recommended: the latter)*
2. Default identifier handling — raw vs hashed+salted? (changes BLE GATT + Firestore schema)
3. Store probe-request SSIDs at all?
4. Exact disclaimer / responsibility split wording for reproducibility + RED.
5. Separate "compliant mode" vs "research mode" in firmware for different jurisdictions?
6. Is sub-GHz (433/868 MHz) in scope at all? If yes, receive-only, and document the elevated legal exposure (see §5). *(recommended: out of scope for v1)*
7. Confirm and ratify "passive RX only, no active TX for scanning" as a hard product principle.

See [`gdpr.md`](gdpr.md) for the data-handling specifics behind decisions 2–3.
