# Agent / Autonomy Model

How the device defines its goals, owns its state, outsources thinking, and
balances its objectives against finite resources. Normative; governed by
[`design_principles.md`](design_principles.md) (esp. P1 and P8).

## 1. Purpose and state (owned locally)

- **Purpose** — user-given objectives/mission plus standing directives
  (e.g. "log BLE anomalies here", "alert on new WiFi networks", "map devices in
  this area over the week").
- **State** — the device's task list/plan, observations, derived knowledge, and
  progress. Held locally; survives with no app and no cloud (P1).
- **Identity** — device keys and configuration (see [`security.md`](security.md)).

The device is always the authority over its purpose and state. Nothing external
owns or overrides them.

## 2. Goal intake

```
User sets a new goal
  → device adds it to its own task list                 (local planning — cheap)
  → OR sends current tasks + new goal to the cloud model
     to generate a new strategy                         (escalation — expensive)
  → device incorporates the result into its own state    (advisory, not authority)
```

The cheap local path is the default. Escalation happens only when the problem is
complex enough *and* both budgets (below) allow it.

Re-planning (the deliberative loop, §5) is triggered not only by a new user goal
but also by **events** — e.g. a radio packet detected by a subsystem, or
accumulated observations crossing a threshold. The device maintains and updates
its own scheduling **policy** in response.

## 3. Outsourced thinking — administratively device-owned

- The device may run heavy reasoning / strategy generation on external compute
  (cloud Gemini), per P3's decentralized, user-owned cloud.
- Those resources are the **device's own project**, under the **user's billing
  account**. Ownership split: **billing = user, project/resources = device.**
- Cloud output is *advisory*: the device merges it into its plan and remains in
  control (P8).

## 4. Resource-bounded planning — two budgets

The device weighs objective value against two finite resources:

| Resource | Budget source | Most expensive action |
|---|---|---|
| **Energy** | power architecture (always-on vs burst domains) | spinning up the burst domain |
| **Money** | cloud billing budget (per period) | cloud escalation |

**Cloud escalation costs both** — money *and* uplink-TX energy — so it is the
most expensive single action and is used sparingly.

Planner inputs (conceptual): objective priority/value, estimated cost per action
(energy; money+energy for cloud), remaining battery vs a reserved floor, remaining
cloud budget for the period, connectivity, and whether the device is on 18650 vs
cold-weather AA primary (different energy headroom).

Floors and guarantees:
- Reserve enough energy for always-on logging and safe shutdown before allowing
  expensive actions.
- Treat the cloud budget for the period as a **hard constraint**, not a target.

## 5. Execution model — control loops, timers, duty cycling

Two tiers keep cost and power down (this is the **locked planner model — option B**:
deterministic execution, LLM for strategy only):

**Reactive loop (always-on, deterministic, cheap).** Runs continuously on the
always-on domain (ESP32-C6 / logging MCU — see [`device.md`](device.md) §2.6).
Executes the current *policy*: timers that duty-cycle subsystems, wake conditions,
thresholds. Reacts to events — a radio packet detected by a subsystem, a timer
firing, a button press, a budget/battery threshold crossed. Powers functions down
during inactivity and wakes the burst domain only when policy says so. This loop
must run and keep the device safe with **no LLM and no cloud** (P1).

**Deliberative loop (burst, occasional, expensive).** Generates and updates the
*policy/strategy* — wake intervals, per-protocol scan duty cycles, escalation
thresholds, quiet hours, budget-aware throttling. Triggered by a new goal or when
accumulated events warrant a rethink. May use local Gemma or escalate to cloud
(P8 / §2). It edits the policy that the reactive loop then executes.

**The schedule/policy is explicit device state** (§1) — data the deliberative loop
writes and the reactive loop reads.

**Events do not automatically wake the expensive domain.** Default handling of an
event (e.g. a detected packet) is cheap always-on logging. Escalation to the burst
domain or cloud is **gated by policy thresholds** (e.g. "wake the compute SoC only
if N anomalous packets within T") — otherwise a busy RF environment would hold the
device fully awake and drain the battery.

**Energy-floor clamp is deterministic.** When battery nears the reserved floor, the
reactive loop clamps to a conservative duty cycle on its own — it does not wait for
or consult the LLM. The deliberative layer only sets policy *within* safe bounds
the reactive loop enforces.

## 6. Cloud-spend governance (safety)

A runaway agent must not be able to drain the user's wallet.

- **GCP Billing Budgets** can be set on the billing account (or scoped to the
  device's project) with alert thresholds. **But budgets only *alert* by default —
  they do not cap spend.**
- **Hard cap requires automation:** budget alert → Pub/Sub → Cloud Function that
  disables billing on the project (or disables the relevant APIs). This is the
  enforcement mechanism, not the budget alone.
- The device should also self-limit: track its own period spend and stop
  escalating before the cap, degrading gracefully to local-only operation (P1
  guarantees it still works).

## 7. Open decisions

1. Goal/utility representation — how objectives are prioritized and how value is
   compared against energy and money cost.
2. Cost model — how the device estimates per-action energy and cloud cost.
3. Escalation policy — concrete thresholds for local vs cloud re-planning, and the
   event-aggregation thresholds (N packets / T window) that gate burst-domain wake.
4. Spend governance — adopt the Pub/Sub→Function hard-cap; define the per-period
   budget and the device's self-limit margin below it.
5. Energy floor — reserved battery percentage and behaviour near it (per sled
   type).
6. Policy/schedule schema — the concrete fields the deliberative loop writes and
   the reactive loop reads (wake intervals, per-protocol duty cycles, thresholds,
   quiet hours).
7. Wake-source wiring — which events/IRQs (radio packet, RTC, GPIO) wake which
   domain, on which processor.

**Resolved (2026-06-16):** planner model = two-tier (option B) — a deterministic
always-on scheduler executes; the LLM generates/updates strategy only. See §5.
