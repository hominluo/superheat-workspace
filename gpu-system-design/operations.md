# Superheat — Operations (Observability/SLOs/Incident, Testing/Simulation, Field Ops & Onboarding)

**Status:** Draft v0.1 · **Date:** June 2026 · **Authority:** OPERATIONAL
**Audience:** SRE, on-call engineering, QA, finance-ops liaison, field-ops, partner/channel management, underwriting/diligence, ABS counsel / backup-servicer, rating-agency pre-read

**Scope:** The full operational surface of the fleet, in three parts that form one loop:

- **Part A — Observability, SLOs, on-call & incident response.** How we *observe* a fleet of up to ~1M GPU nodes embedded in water heaters, what we promise about it (SLOs), how we get paged and what we can actually *do* given that a node is outbound-only behind CGNAT and may be physically unreachable for months, how we respond when things break, and how the control plane survives (DR/HA) and is escrow-able (backup servicer).
- **Part B — Testing & simulation.** How we validate the system *before* it touches a physically-unreachable, safety-critical node: the test pyramid, contract conformance, the fleet-scale simulator, control-system validation, the HIL safety rig, chaos/fault-injection, and the release gates.
- **Part C — Field operations & onboarding.** How a node gets certified-installed, identity-bootstrapped, serviced, GPU-swapped at year-5, and recovered when a homeowner stops being a willing host; plus customer onboarding.

**These three parts are one system.** Part A's SLOs are precisely what Part B's tests assert against and what Part A's incidents violate; Part C generates the physical units that Part A then monitors and Part B's HIL rig mirrors. They are merged here so the loop is visible: *field-ops originates collateral → testing gates the software that reaches it → observability watches it earn → incident response and recovery protect both safety and the tape.*

## Governing principles (shared by all three parts)

Two facts about this fleet shape every runbook, every test, and every install procedure below:

1. **The control plane never dials a node.** Every diagnostic, every mitigation, every console session, every install handshake is *node-initiated outbound* over the WireGuard overlay (inherited from the node and connectivity designs, see `node.md` and `connectivity-and-data.md`). An on-call engineer's action surface — and an installer's — is bounded by what an already-connected agent will do; a node that is offline is beyond reach until it dials back in or someone drives to it. This is why a single offline node is normal (not an incident), why testing is the *only* way software reaches a node, and why an installer must bring a node online **without ever touching the customer's router**.

2. **Nodes are fail-safe-to-heater and safety-critical: they must never lose hot water.** Safety is *local* — the thermal MCU is the safety authority and keeps heating regardless of cloud, agent, or GPU state. A zero-hot-water regression is not a bug; it is a broken customer relationship and a brand event. This is the hard **zero-hot-water SLO (S1)** in Part A, the **HIL safety suite + release gate** in Part B, and the **heater-first SLA** in Part C — *stated once here, referenced throughout.*

A consequence that runs through the whole doc: **a servicing/cloud/install gap is an earning gap, never a safety gap.** Even a total servicer outage leaves a million working water heaters; the asset degrades to "doesn't earn," not "dangerous."

**Two budgets, deliberately separate (the authoritative-data rule of `platform.md`):** telemetry/health SLOs are *lossy and approximate*; metering/tape SLOs are *exact and money-grade*. Never trade one budget for the other. A telemetry outage burns the observability budget, not the tape's correctness guarantee. Part B tests this separation; Part A is on-call against both.

This doc consolidates SLOs that were scattered per-service in `platform.md` and adds the fleet-level SLIs that no single service owns. It builds on the canonical telemetry frame and reliability score in `data-contracts.md` (the source of node-level SLIs); it does not redefine them. It also de-risks **R7 (servicer concentration / backup servicer)** from `financial-and-roadmap.md`: the Part A runbooks *are* the escrow deliverable.

---
---

# PART A — Observability, SLOs, On-Call & Incident Response

---

## 1. Consolidated SLO catalog

SLIs are split into three planes: **node-level** (per-node, aggregated to fleet percentiles), **fleet-level** (cross-node properties no single service owns), and **control-plane** (the cloud services in `platform.md`). Error budgets are monthly unless noted. The node-level SLIs are computed from the canonical telemetry frame (`data-contracts.md` §5) and feed the reliability score (`data-contracts.md` §3).

| # | SLI | Plane | Target (SLO) | Monthly error budget | Source / owner |
|---|---|---|---|---|---|
| S1 | **Zero-hot-water incidents** attributable to compute/OTA/agent | Node (safety) | **0 incidents / month, fleet-wide** (hard SLO) | **0** — any breach is a Sev-1 and a standing release-gate failure (R9) | thermal-fault telemetry + truck-roll tickets; node thermal control (`node.md`) |
| S2 | Telemetry ingest availability (node→hot-tier) | Control-plane | ≥ 99.9% | 43 min/mo | edge aggregator → ingest (`platform.md`) |
| S3 | Telemetry visibility latency (node→hot-tier) | Control-plane | p95 < 60 s | 5% of frames may exceed | aggregator decimation lag (`platform.md`) |
| S4 | Alert evaluation lag (1-min tier) | Control-plane | p99 < 30 s | 1% may exceed | alert evaluator (`platform.md`) |
| S5 | Scheduler placement decision latency | Control-plane | p95 < 2 s for available capacity | 5% may exceed | scheduler (`platform.md`) |
| S6 | Control-plane (regional) availability | Control-plane | ≥ 99.95% per region | 22 min/mo/region | scheduler+registry+ledger ingest API |
| S7 | Wrongful non-interruptible preemption | Control-plane | < 0.5% of premium placements | — | scheduler (`platform.md`); `data-contracts.md` §7 predicate |
| S8 | Idle-window leak (usable window, no job >5 min, backlog present) | Fleet | < 1% of window-minutes | — | scheduler + thermal coordinator (`platform.md`) |
| S9 | Reliability score freshness | Control-plane | recomputed ≥ daily/node; tier change → scheduler cache < 5 min | — | reliability service (`platform.md`) |
| S10 | **Tape (metering) coverage** | Control-plane (money) | 100% of paid GPU-hours backed by a double-signed `usage_record` | **0** — any uncovered paid hour blocks settlement | ledger (`platform.md`; `data-contracts.md` §8) |
| S11 | **Tape reconciliation freshness** | Control-plane (money) | three-way reconcile (usage↔marketplace↔bank) drift ≤ ±2%, opened < 1 reporting period after period end | drift > 2% ⇒ audit ticket + settlement hold (R8) | ledger reconciliation (`platform.md`) |
| S12 | Monthly tape close | Control-plane (money) | < 24 h after period end, fully reproducible from append-only `usage_record` | — | tape view (`platform.md`) |
| S13 | OTA auto-halt latency after gate breach | Control-plane | < 10 min | — | OTA orchestrator (`platform.md`) |
| S14 | OTA rollback success (bricked/failed node self-recovers w/o truck roll) | Node | ≥ 99.5% of rollouts; thermal-faults attributable to OTA = 0 | thermal-fault = **0** (hard, ties S1) | A/B auto-rollback (`node.md`); OTA gates (`platform.md`) |
| S15 | OTA rollout pace (clean release canary→fleet) | Control-plane | completes < 14 days (slow by design) | — | OTA staged rollout (`platform.md`) |
| S16 | **Per-node uptime** (heartbeat fraction, excl. cloud-down windows) | Node | premium ≥ 99.0% / standard ≥ 97% / spot ≥ 90% (tier-banded) | tracked per node into score | telemetry `seq` gaps (`data-contracts.md` §3, §5) |
| S17 | **Per-node thermal availability** (could-run-when-asked) | Node | premium ≥ 70% / standard ≥ 55% / spot best-effort | tracked per node into score | thermal coordinator window history (`platform.md`) |
| S18 | Window prediction error (predicted vs actual usable seconds) | Fleet | p50 within ±15% | — | thermal coordinator (`platform.md`) |
| S19 | Aggregator buffer depth during cloud outage | Fleet | ≥ 2 h of telemetry buffered, zero loss for transient outages | — | edge aggregator (`platform.md`) |
| S20 | Control-plane DR | Control-plane | **RTO ≤ 30 min, RPO ≤ 5 min** for scheduler/registry/ingest; **RPO = 0** for the ledger | — | §5 below |

**Cross-references that retire local restatements:** S2–S4 supersede the scattered telemetry-pipeline numbers in `platform.md`; S5/S7/S8 consolidate the scheduler SLOs; S9 consolidates the reliability-service SLO; S10–S12 consolidate the tape SLOs; S13–S15 consolidate the OTA SLOs; S17/S18 consolidate the thermal-coordinator SLOs. Those sections of `platform.md` remain authoritative for *mechanism*; this table is authoritative for the *numbers we are on-call against*. These same numbers are what **Part B** asserts in simulation, HIL, and the release gate.

**Note on the cloud-down exclusion (the false-outage fix):** S16 uptime explicitly *excludes* heartbeat gaps stamped `cloud_degraded` by the aggregator (`data-contracts.md` §3, §5 telemetry field). A correlated cloud outage must never burn nodes' uptime budgets or drop their reliability tier — that would corrupt the tape (R8). This is also why the alerting taxonomy below distinguishes node-down from cloud-down with a hard correlation test (§3.3), and why Part B's reliability-score replay test asserts the exclusion (see Part B).

---

## 2. Observability stack

### 2.1 The three signals at fleet scale

```
NODE (node.md)                               EDGE (regional)               CLOUD
─────────────────────────────────            ──────────────────           ──────────────────────────
metrics  telemetry frame @10 s  ──overlay──► [Edge Aggregator] ──1-min──►  [TSDB hot/warm/cold]
         (data-contracts.md §5)              decimate, attest-check,       dashboards, alert eval
                                             buffer ≥2 h, cloud_degraded    reliability scoring (§4)
logs     agent + job logs, ring-buffered     ship ERROR/WARN + gate         [Log store] sampled;
         on STATE; full set local 24 h       results + rollback events      full pull only via console
traces   control RPCs (place/dispatch/       n/a (node has no tracer)       [Trace store] cloud-side
         attest) cloud-side only                                            request spans only
```

- **Metrics are the spine.** The canonical telemetry frame (`data-contracts.md` §5) is the per-node SLI source: `gpus[].util_pct/temp_c/power_w/xid_errors`, `thermal.mode/heat_demand_w/tank_temp_c/comfort_ok`, `board_power_w`, `net.path/uplink_mbps/rtt_ms/loss_pct`, `ckpt_age_s`, `seq` (gap = lost samples → uptime SLI S16), and `cloud_degraded`. Every node-level SLI in §1 is a function of these fields. The frame is signed (`sig`) so a metric can't be spoofed by a compromised relay.
- **Logs are pull-not-push at fleet scale.** Nodes ring-buffer full logs locally (24 h, `STATE`/`CACHE` partitions per `node.md`) and only *ship* ERROR/WARN, OTA gate results, rollback events, and state-machine transitions. The full log set for a specific node is fetched on demand over the console tunnel during an incident (Rung 3, `node.md`) — you cannot stream verbose logs from 1M residential uplinks.
- **Traces are cloud-side only.** We trace the control-plane request path (yield-router → scheduler `place()` → thermal `GetWindow` → dispatch → ledger sign) because that's where latency SLOs S5/S6 live. The node does not run a distributed tracer; its "trace" is the telemetry `seq` + state-machine transitions.

### 2.2 Cardinality: what you can and cannot store per node

At ~1M nodes a naïve "every field, every node, every 10 s, forever" is ~100K frames/s and astronomical cardinality. The rule:

| You CAN store… | You CANNOT store (without aggregation) |
|---|---|
| Per-node **1-min rollups** in hot TSDB for 30 d (S3 tier) | Per-node **10-s raw** centrally — kept only at the edge for 24 h (incident replay) |
| Per-node scalar gauges (temp, util, power, tank_temp, uplink) | Per-(node × GPU × per-second) high-cardinality series fleet-wide |
| **Fleet/region/SKU/tier-bucketed** histograms and counters (the dashboards run on these) | Per-node log lines centrally (pulled on demand only, §2.1) |
| Per-node daily reliability snapshot (`node_reliability_history`, `platform.md`) | Per-node high-frequency event streams (debounce/aggregate at edge) |

**Edge aggregation is mandatory, not an optimization** (`platform.md`). The regional edge aggregator: terminates node overlay streams, validates the attestation token on the frame, decimates 10 s → 1 min for the cloud, keeps 10 s locally for 24 h, and **buffers ≥ 2 h** so a cloud-ingest outage causes zero telemetry loss (SLI S19) and the nodes never block on the cloud (they keep heating and running already-dispatched jobs).

### 2.3 Dashboards (the on-call's working set)

1. **Fleet health** — count by `lifecycle_state` (`platform.md`), by reliability tier (`data-contracts.md` §2), online vs `cloud_degraded` vs node-down; the S16/S17 distributions. *Top tile: live count of nodes in `HEATER-ONLY SAFE-MODE` — the S1 leading indicator.*
2. **Telemetry pipeline** — ingest rate, aggregator buffer depth (S19), decimation lag (S3), alert eval lag (S4), per-region.
3. **Scheduler/yield** — placement p95 (S5), idle-window leak (S8), preemption rate (S7), backlog depth.
4. **OTA** — active rollouts by `stage`, per-cohort `applied/rolled_back/failed` rates, gate-breach status, halt state (S13–S15).
5. **Thermal/safety** — `comfort_ok=false` count, tank-temp excursions, `heater_safe=false` events, OTA-attributable thermal faults (S1/S14 — must read zero).
6. **Tape/money** (finance-ops co-owned) — usage-record coverage (S10), three-way reconciliation drift (S11), close status (S12). *Read-only for on-call; finance-ops owns mitigation, but on-call is paged on the breach.*

---

## 3. Alerting taxonomy & on-call

### 3.1 Severities

| Sev | Definition | Response | Pages? | Examples |
|---|---|---|---|---|
| **Sev-1** | Customer safety, fleet-wide earning capability, or tape integrity at risk | Immediate page, war-room, exec/finance notify | **Page** (24/7) | Any zero-hot-water incident (S1); control-plane regional outage gating attestation/earning (S6); bad OTA causing thermal faults (S14); tape reconciliation breach that blocks settlement (S11) |
| **Sev-2** | Significant degradation, SLO burn, no safety/money impact yet | Page during business + escalate if burning fast | **Page** | Telemetry ingest down for a region (S2); scheduler placement p95 blown (S5); mass-offline that is *real* (not correlated cloud-down); OTA gate breach + auto-halt fired (S13) |
| **Sev-3** | Localized / single-node / slow burn | Ticket; batch-handle | Ticket | Single node flapping; one node stuck post-OTA; idle-window leak above S8 in one region |
| **Sev-4** | Informational / capacity / trend | Ticket / dashboard | Ticket | Reliability-tier drift in a cohort; relay-cost trend (R2); seasonal thermal-availability dip (R5) |

### 3.2 What pages vs what's a ticket — the page-gate

A node going offline is **normal** for this fleet (homes lose power, ISPs blip, people unplug heaters). **A single offline node never pages** — it's a Sev-3/4 reliability-score input, not an incident. The page-gate is:

- **Correlation:** page only when the offline/degraded set exceeds a region- and time-of-day-adjusted baseline (residential nodes have diurnal patterns) **and** the affected count crosses a threshold (e.g., >2% of a region in <10 min).
- **Safety always pages**, regardless of count: any `comfort_ok=false` / `heater_safe=false` / OTA-attributable thermal fault is Sev-1 even for one node (S1 is a *zero* SLO).
- **Money always pages** at the breach line: S10/S11 breaches are Sev-1 even though finance-ops, not on-call, drives the fix.

### 3.3 Node-down vs cloud-down (correlated-outage handling) — the most important alert

Before paging "mass node offline," the alert evaluator runs a **correlation test** to decide whether the *cloud* is blind or the *nodes* are gone — because the mitigation, the blast radius, and the budget hit are completely different.

```
on (offline_count spikes):
  if  ingest_rate_drop is region-wide AND aggregator buffers are FILLING (not draining)
      AND control-plane health checks degrade
   -> CLOUD-DOWN / OBSERVABILITY OUTAGE   (we are blind; nodes are probably fine)
      * aggregator stamps cloud_degraded=true on the gap window (data-contracts.md §5)
      * those gaps are EXCLUDED from uptime SLI S16 / reliability score (data-contracts.md §3)
      * page Sev-2 (Sev-1 if it gates attestation/earning, §4 runbook)
      * DO NOT mass-quarantine nodes; DO NOT trigger OTA rollback
  elif offline nodes are geographically/ISP-correlated but cloud is healthy
   -> EXTERNAL OUTAGE (ISP/power); nodes safe-mode locally, will dial back
      * Sev-2/3 by scale; exclude from score if attributable to a known ISP/grid event
  else (scattered, cloud healthy, real node failures)
   -> REAL NODE FAILURES; Sev-2 if mass, Sev-3 if isolated; proceed to triage
```

The `cloud_degraded` stamp is the mechanism that prevents a cloud blip from corrupting the tape (R8) and unfairly demoting nodes (S16 exclusion). This exact correlated-outage behavior is fault-injected and asserted in **Part B** (chaos: correlated cloud/aggregator outage).

### 3.4 What an on-call engineer can ACTUALLY do to a node

This is the hard constraint (governing principle 1). The action surface, cheapest → last-resort, mapping to the node recovery ladder (`node.md`):

| Action | Reachable when… | Mechanism | Cannot do |
|---|---|---|---|
| Read telemetry/score | always (lossy if degraded) | TSDB / dashboards | — |
| **Remote console** (logs, inspect, push fix) | node is **dialed in** to overlay | Rung 3 reverse tunnel over WireGuard (`connectivity-and-data.md`, `node.md`) — outbound-only | reach an offline node |
| **OTA / push fix image** | node dialed in + thermal window | enqueue target digest; agent applies A/B on its own schedule (`platform.md`) | force-apply instantly; bypass health gate |
| **A/B rollback** | node dialed in OR self-triggers | enqueue rollback-to-previous-slot; or node auto-rolls-back locally with **no cloud at all** (`node.md`) | — (this is the strong card) |
| **Quarantine** (stop paid work) | node dialed in | set `lifecycle_state=quarantined`, scheduler stops dispatch | stop a node that's already offline (it's already not earning) |
| **Force heater-only safe-mode** | node dialed in | command agent; or it's already there locally | — |
| **Truck roll** (physical) | last resort | dispatch installer/technician (Part C, §4) | be fast or cheap; node may be unreachable for months |

The on-call's job is to **maximize what the recovery ladder fixes without a truck roll** and to *correctly classify* the rare cases that genuinely need one. A node in `HEATER-ONLY SAFE-MODE` is *not urgent* from a safety standpoint — the customer has hot water — so it can wait for a batched truck roll; it's urgent only as lost earning capacity. (Truck-roll dispatch, RMA, and the remote-vs-truck-roll decision matrix are Part C, §4.)

### 3.5 Escalation

`page → primary on-call (ack ≤ 5 min) → secondary (ack ≤ 15 min) → SRE lead → eng lead`. Sev-1 additionally notifies: **safety incidents →** product/legal + (if pattern) field-ops for truck rolls (Part C); **tape/money →** finance-ops liaison + (if material) ABS counsel/backup-servicer per R7. War-room opens for any Sev-1 and any Sev-2 burning the budget faster than 1×.

---

## 4. Incident response runbooks

Format: **Detection → Triage → Mitigation → Comms.** Three are fully numbered; the rest are condensed. All assume the §3.4 action surface — no action exists that dials a node.

### 4.1 Runbook — Mass node offline (correlated)

**Detection:** offline/degraded count crosses the §3.2 page-gate for a region; Sev-2 page.

1. **Run the correlation test (§3.3) FIRST.** Open the telemetry-pipeline dashboard: are aggregator buffers *filling* (cloud blind) or are nodes genuinely gone?
2. **If buffers filling / cloud-side ingest degraded → this is CLOUD-DOWN.** Confirm aggregator stamped `cloud_degraded=true`. **Do not** mass-quarantine, **do not** trigger OTA. Jump to Runbook 4.4 (control-plane outage). Verify the `cloud_degraded` window is being excluded from reliability scoring (S16) so the tape isn't corrupted.
3. **If geo/ISP-correlated and cloud healthy → EXTERNAL (ISP/power) outage.** Cross-check ISP status pages / grid-event feeds. Nodes are in local safe-mode and will dial back; tag the window for score exclusion. No node action needed.
4. **If scattered + cloud healthy → REAL failures.** Bucket by SKU / `agent_version` / `os_slot` / recent rollout cohort. A spike concentrated in one `agent_version` or rollout cohort means **go to Runbook 4.2 (bad OTA)** — this is the most likely "real" mass-offline cause we can act on.
5. **Mitigate what's reachable:** for nodes that *are* dialed in but unhealthy, console-inspect a sample (Rung 3); if a bad image is implicated, enqueue rollback. Nodes that are *offline* cannot be touched — record them, let auto-recovery (watchdog → A/B rollback) work, and only schedule truck rolls (Part C, §4) for nodes that stay dark beyond the SLA and are confirmed not ISP/power.
6. **Comms:** post incident channel with the *classification* (cloud/ISP/real) up front — that's what stops people from panic-quarantining. If earning is affected, notify finance-ops (impacts utilization tape). Status page only if customer-facing (it usually isn't — customers still have hot water).

### 4.2 Runbook — Bad OTA rollout

**Detection:** OTA health-gate breach (`platform.md`) — boot-success < 99.5%, crash/watchdog rate >1.5× baseline, elevated GPU faults, or **any new thermal-safety fault** (hard stop); or a fleet-wide spike in node-reported `rolled_back`. Sev-1 if thermal faults present (ties S1/S14), else Sev-2. Auto-halt should already have fired (S13, < 10 min).

1. **Confirm auto-halt fired.** Rollout `stage` should be `halted`. If not, manually halt the rollout NOW (set `stage=halted`) — stop the bleeding before diagnosing.
2. **Check the safety line first.** Query thermal-fault telemetry for the rollout cohort. **Any** OTA-attributable `comfort_ok=false` / `heater_safe=false` is a Sev-1 zero-tolerance breach (S1) — escalate immediately, this is the R9 release-gate failure.
3. **Verify rollback is propagating.** The orchestrator should have enqueued rollback-to-previous-slot for all `applied` nodes and set them `quarantined` (`platform.md`). Watch `ota_node_status.result` flip to `rolled_back`. Nodes that boot-fail locally auto-roll-back with no cloud help (`node.md`) — these self-heal; the cloud just records it.
4. **Identify the un-rolled-back tail:** nodes that took the bad image but are now **offline** can't be commanded. They will hit their local boot/watchdog rollback (Rung 1→2) on their own. Do not truck-roll until the local ladder has demonstrably been given time to fire.
5. **Root-cause the gate miss:** the field hit something CI's HIL rig (Part B, §11–§13) didn't — driver/hardware variance, thermal-link timing, uplink behavior. File the gap into the Part B regression suite so the next image can't repeat it.
6. **Comms:** incident channel with cohort scope, rollback progress %, and an explicit "customer hot-water impact: yes/no." If yes → Sev-1 path (§3.5). Do not resume the rollout until a fixed image passes an expanded gate on canary again (S15 pace is intentionally slow).

### 4.3 Runbook — Thermal / safety fault (zero-hot-water risk)

**Detection:** telemetry `comfort_ok=false` or `heater_safe=false`, tank-temp below comfort floor, or a node reporting `HEATER-ONLY SAFE-MODE` with the *element also faulted*. **Always Sev-1, even for one node** — S1 is a hard zero SLO. This is the one place a single node pages.

1. **Distinguish the two cases.** (a) Node in `HEATER-ONLY SAFE-MODE` with the **conventional element working** = customer HAS hot water, compute is off → this is the *designed* fail-safe (`node.md`), **not** an S1 incident; downgrade to Sev-3 (lost earning, batched truck roll). (b) Element faulted / tank below comfort and not recovering = **real** S1.
2. **For a real S1:** the node's local control loop and MCU (`node.md`) are the safety authority and have already cut GPU and prioritized heating — verify `heater_safe` semantics are being honored in telemetry. The cloud cannot make it safer; the cloud's job is to confirm and dispatch help.
3. **Force compute off** on the node and any peers in the same cohort if a systemic cause is suspected (set `quarantined`, command safe-mode) — never let compute compete with heat during a safety event.
4. **Check for a systemic signal:** is this one bad heater (truck roll) or a pattern across an `agent_version`/SKU (then it's also Runbook 4.2)? A pattern is far more serious than a one-off.
5. **Dispatch field service** for the affected unit(s) — this *is* a truck roll (Part C, §4); safety overrides the no-truck-roll preference. Track to resolution; it cannot close until the customer has hot water restored (the heater-first SLA, Part C, §4.3).
6. **Comms:** Sev-1 page, notify product/legal (customer relationship per R9), and finance-ops if it touches a financed unit's earning. Every S1 is a standing release-gate failure — the offending release/cohort is frozen fleet-wide until root-caused (Part B, §13).

### 4.4 Runbook — Control-plane outage (fleet-wide earning blast radius)

**Detection:** S6 regional availability breach; scheduler/registry/ingest health-checks fail; aggregator buffers filling. **Sev-1**, because **attestation gates earning** (`node.md`): if the control plane can't attest nodes or counter-sign usage records, the *whole region stops earning* even though every node is physically fine and still heating.

- **Triage:** identify which service is down (registry/ingest/scheduler/ledger). Confirm nodes are *safe* — they are: outbound-only + fail-safe means nodes keep heating, keep running already-dispatched jobs, and **buffer usage records locally** to reconcile on reconnect (`platform.md`). No safety impact; this is purely an earning/observability outage.
- **Mitigation:** execute §5 DR. Restore the *ingest + attestation + scheduler* path first (that's what unblocks earning); the tape close (S12) can lag, it's reconstructable from append-only `usage_record`. Ensure the outage window is stamped `cloud_degraded` so it's excluded from every node's uptime/score (S16) — **the blast radius must not become a fleet-wide reliability-tier demotion.**
- **Comms:** Sev-1, finance-ops immediately (earning is paused → utilization tape dip that must be explained as cloud-attributable, not node-attributable — directly relevant to R8 tape credibility). Backup-servicer/ABS counsel if outage exceeds the deal's servicing-continuity threshold (R7).

### 4.5 Runbook — Tape reconciliation breach (condensed)

**Detection:** S11 drift > ±2% in any leg (usage↔marketplace↔bank) or S10 uncovered paid hours. **Sev-1**, finance-ops-led, on-call paged.
**Triage:** which leg drifted — connector/price drift (marketplace leg, `platform.md`), missing/duplicate usage records (S10), or settlement/bank mismatch. Check the per-GPU join is correct (`data-contracts.md` §8 — the historical Cartesian-join bug).
**Mitigation:** open audit ticket, **hold settlement** for affected nodes/tranche (`platform.md`), reconstruct from append-only `usage_record`. Telemetry is *not* the arbiter here — money comes from signed usage records, not lossy telemetry (`data-contracts.md` §8).
**Comms:** finance-ops + ABS counsel; this is the R8 "kills deals" risk — every breach gets a written variance report (quarterly static-pool discipline). The reconciliation harness that *prevents* this is Part B, §17.

### 4.6 Runbook — Security incident (condensed)

**Detection:** failed attestation cluster, anomalous tenant egress (mining-pool / home-LAN attempts blocked by the egress firewall, `node.md`), key-compromise indicators, or spoofed/invalid telemetry signatures. Sev-1 if attestation/identity integrity or homeowner-LAN safety is implicated.
**Triage:** scope — single tenant, single node, or fleet (e.g., leaked signing path). Attestation failures *contain themselves*: a node that can't attest gets no paid work but still heats (`node.md`), so the blast radius on identity issues is bounded by design.
**Mitigation:** quarantine affected nodes (stop paid work), revoke/rotate compromised credentials, force re-attestation/re-image via OTA, tighten egress allowlists. A stolen disk yields nothing (secrets sealed to TPM + image measurement, `node.md`; see also `security-and-compliance.md`).
**Comms:** security lead + legal; if homeowner LAN was reachable, treat as customer-trust Sev-1.

---

## 5. Control-plane DR / HA

The control plane is the single point of failure for the fleet's *ability to earn* (not its safety — safety is local). RTO/RPO targets are SLI **S20**.

| Component | HA posture | RTO | RPO | Backup / failover |
|---|---|---|---|---|
| **Registry + Ledger (Postgres/Supabase)** | Primary + sync standby, multi-AZ; per-region shard (`platform.md`) | ≤ 30 min | **0** (synchronous replication; money path) | PITR + continuous WAL archiving; append-only `usage_record` is reconstructable (S12) |
| **Telemetry ingest + TSDB** | Stateless writers; aggregator buffers ≥ 2 h (S19) | ≤ 30 min | ≤ 5 min (lossy tier is acceptable) | Kafka/Redpanda retention; cold Parquet (`platform.md`) |
| **Scheduler / yield / scoring** | Stateless workers behind queues, per region | ≤ 15 min | ≤ 5 min (state in Postgres/Redis) | redeploy workers; node-state cache rebuilds from telemetry |
| **OTA orchestrator** | Stateless; rollouts persisted in Postgres | ≤ 30 min | 0 (rollout state in DB) | resume from `ota_rollout`/`ota_node_status` |
| **Overlay coordination (headscale/Tailscale)** | Multi-region coordination servers + relay/TURN pool | ≤ 15 min | n/a | nodes reconnect-with-backoff (`connectivity-and-data.md`); they tolerate coordinator restarts |

**Multi-region:** the architecture is already region-sharded (`platform.md`) — a node only ever talks to its regional plane. This bounds any single-region outage to that region's earning; it is **not** a global fleet-down event. The tape fans in per region at period close, so one region's DR does not block another's settlement. Critical invariant from §3/§4: **a control-plane outage must stamp `cloud_degraded` so it never demotes node reliability or corrupts the tape** (R8) — DR is not just availability, it's tape-integrity protection.

**DR drills:** quarterly region-failover game-day; the backup-servicer failover drill (R7, below) is run before the inaugural ABS (Phase C exit, see `financial-and-roadmap.md`). Fault injection on these paths is owned by Part B (§16 chaos / §11 release gates).

---

## 6. Backup-servicer / escrowed-runbook angle (R7)

The rating committee's objection is **servicer concentration**: "only Superheat can run this fleet." `financial-and-roadmap.md` R7 names the standard esoteric-ABS cure — a named backup servicer plus escrowed orchestration runbooks/code. **Part A of this document is the human-readable half of that escrow deliverable.**

### 6.1 What makes these runbooks the escrow deliverable

- **They are operational, not aspirational.** §1's SLO catalog tells a replacement servicer exactly what "running the fleet correctly" means as numbers; §3–§4 tell them what to watch and what to do; §5 tells them how to keep the plane alive. That is the servicing manual.
- **They encode the constraints that make this fleet weird.** A generic SRE handed a normal fleet would try to SSH in, dial nodes, and aggressively quarantine on mass-offline — all wrong here. §3.3 (correlated-outage handling) and §3.4 (what you can actually do behind CGNAT) are precisely the non-obvious knowledge a backup servicer must have to *not destroy the tape* on day one.
- **They protect the collateral, not just uptime.** The repeated `cloud_degraded` / score-exclusion discipline and the two-budget separation (telemetry vs money) exist so that a servicing transition does not corrupt the reliability track record or the reconciliation that the ABS is rated on (R8).

### 6.2 What a replacement servicer needs (the escrow package)

1. **This doc + the design docs it references** (`data-contracts.md` for the canonical schemas, `platform.md` for the services, `node.md` for node behavior, `connectivity-and-data.md` for the overlay, `financial-and-roadmap.md` for the risk register). Note that the certified installer/T3 network (Part C, §1) is *itself* a backup-servicer asset — a deep field network is part of the cure.
2. **Credentials & key custody** — control-plane signing keys / counter-signing key custody, overlay coordination admin, registry/ledger DB access, OTA signing keys (held in escrow, released on a servicing-transfer trigger).
3. **The "first 72 hours" priority order:** (a) restore attestation + ingest + scheduler so the fleet earns (Runbook 4.4); (b) verify tape reconciliation and the two-budget separation (Runbook 4.5); (c) confirm the safety floor is intact (every node can still fall back to heater-only locally regardless of cloud state — the comforting fact a backup servicer inherits: **no node depends on the servicer to be safe**).
4. **The failover drill record** — the tested servicer-failover game-day (R7 Phase-C exit) proving a third party can stand up the plane from escrow within RTO.

The single most reassuring sentence for the rating committee, and the design fact that makes a backup servicer credible: **a servicing gap is an earning gap, never a safety gap** (governing principle 2). Because every node is fail-safe-to-heater and outbound-only, even a total servicer outage leaves a million working water heaters — the asset degrades to "doesn't earn," not "dangerous." That bounds the backup servicer's worst case and is why the escrowed runbook is a sufficient cure.

---
---

# PART B — Testing & Simulation Strategy

---

**Why this part is load-bearing.** Two facts make testing here unusually consequential (they are the governing principles restated for the test author):

1. **Nodes are physically unreachable for months.** A bad behavior caught in the field costs a truck roll or a dead asset; there is no SSH-in-and-fix. CI is *the only way software reaches a node* (`node.md`), so the test pyramid is the production safety net, not a courtesy.
2. **Nodes are safety-critical: they must never lose hot water** (R9). A regression here is not a bug, it is a broken customer relationship and a brand event.

Everything below is built to gate those two facts. The control systems are validated *in simulation* before they are promised to financiers, and *in hardware-in-the-loop* before any OTA reaches a real tank. The scheduler, the thermal coordinator, the yield router, and the metering tape (all in `platform.md` / `data-contracts.md`) are the moat — and the harness specified here is what validates them. These tests assert against the **Part A SLO catalog** (S1–S20) and produce the gates that Part A's incident runbooks reference.

The canonical schemas/predicates we test against are NORMATIVE and live in `data-contracts.md`.

---

## 7. The test pyramid

We test the way the system is built: a wide base of fast deterministic checks, a contract-conformance band that the *entire* system depends on (because every service speaks the `data-contracts.md` lingua franca), a fleet-scale simulator that is unusually large for our pyramid (because the moat is a control system over thousands of nodes), HIL for the safety-critical edge, and a thin end-to-end + chaos cap.

```
                          ▲  slower, fewer, higher-fidelity, costlier
                          │
            ┌─────────────────────────────────┐
            │   E2E + CHAOS (§13, §16)         │   real cloud + HIL rig + canary fleet;
            │   release gates; fault injection │   correlated outage, bad OTA, XID storms
            ├─────────────────────────────────┤
            │   HARDWARE-IN-THE-LOOP (§12)     │   1 bench/region: OS, agent, A/B rollback,
            │   safety interlock, recovery     │   recovery ladder, thermal MCU fail-safe
            ├─────────────────────────────────┤
            │   FLEET-SCALE SIMULATION (§9,§10)│   discrete-event sim of 1k–100k nodes;
            │   scheduler/router/thermal SLOs  │   validates p95<2s, <1% idle-leak, 60%×70%
            ├─────────────────────────────────┤
            │   CONTRACT CONFORMANCE (§8.3)    │   every producer/connector round-trips the
            │   data-contracts.md round-trip   │   canonical schemas + predicates (NORMATIVE)
            ├─────────────────────────────────┤
            │   INTEGRATION (§8.2)             │   service↔service, DB txns, replay against
            │   real Postgres/Redis/Kafka      │   recorded telemetry
            ├─────────────────────────────────┤
            │   UNIT (§8.1)                    │   pure functions: score formula, fits(),
            │   deterministic, ms-fast         │   duty-cycle math, period-split, EV math
            └─────────────────────────────────┘
                          │  faster, many, lower-fidelity, cheap
                          ▼
```

A change cannot promote past a band until the band below is green for the touched components. The simulator (§9) is deliberately a *first-class tier*, not an afterthought, because the things that pay for the company — placement SLO, utilization, tape — only emerge at fleet scale.

## 8. Lower tiers: unit, integration, conformance, E2E

### 8.1 Unit tier

Pure-function tests on the math the moat is made of. No I/O, deterministic, sub-millisecond, run on every commit. Mandatory unit coverage targets:

| Function under test | Source of truth | Representative cases |
|---|---|---|
| `reliability_score(...)` | `data-contracts.md` §3 | perfect node → **exactly 100** (the previous formula summed to 0.80 — this is a regression guard); fault_penalty caps at −30%; `cloud_degraded` windows excluded from `uptime_frac`; seasoning gate forces `probation` regardless of score |
| `tier_order()` comparison | `data-contracts.md` §2 | `premium>standard>spot>probation`; `trusted_host` never appears in `reliability_tier_required` |
| `fits(job, node, window_remaining_s)` | `data-contracts.md` §7 | non-interruptible job with `expected_duration_s=None` → **false**; interruptible needs `max(min_thermal_window_s, T_GRACE_S=30)`; gpu_count > free_gpus → false; `trusted_host` mismatch → false |
| thermal `C_th`, `E_store`, `storage_headroom_kwh` | `data-contracts.md` §6 | `E_store ≈ 8.8 kWh` for 80-gal H1; per-SKU `usable_thermal_kwh` (the hardcoded `12.0` is **forbidden** — assert it never appears); headroom clamps ≥ 0 |
| `thermal_window()` runnable-minutes | utilization engine (`platform.md`) | discharge-led vs charge-led; tank clamps `[T_min,T_max]`; `D(t)` makes room |
| `preheat_start()` | utilization engine (`platform.md`) | latest charge start covers forecast peak with safety margin |
| yield-router `E[$/gpu-hr]` | yield router (`platform.md`) | worked example reproduces table (vast=0.227, runpod=0.167…); `net = gross·(1-fee)·(1-fx)`; served_fraction clamp for window < runtime |
| `apply_concentration_guard()` | yield router (`platform.md`) | at 62% share, vast EV ×0.5 → RunPod wins; `direct/reserved/backlog` exempt; graduated penalty between warn(0.50) and cap(0.60) |
| period-boundary split | `data-contracts.md` §8 | a usage record **never** spans a month boundary; long job → one record per period |

### 8.2 Integration tier

Service-against-service, with real backing stores spun up in containers (Postgres/Supabase, Redis, Kafka/Redpanda — the `platform.md` "buy" substrate). No mocks for the money path.

- **Scheduler ↔ registry ↔ thermal coordinator:** `place(job)` reads node metadata (Redis cache + Postgres), hard-filters on a live `ThermalWindow`, reserves GPU rows in a transaction; assert no double-booking under concurrent `place()` calls (the `UNIQUE (node_id, slot_index)` and `gpu.state` transition).
- **Telemetry ingest → scoring → tier propagation:** push `superheat.telemetry.v1` frames through the edge-aggregator decimation, assert a tier change propagates to the scheduler cache **< 5 min** (`platform.md` SLO; Part A S9).
- **Connector → normalized job → scheduler:** each `MarketplaceConnector.normalize()` output is fed to the scheduler unmodified (proves the lingua franca actually type-checks across the boundary — the original sin §8.3 exists to prevent).
- **Usage record → ledger → tape view:** insert double-signed records, refresh `securitization_tape`, assert the GPU-join is on **both** `node_id` AND `gpu_slot` (the Cartesian-multiply bug guard, `data-contracts.md` §8).
- **OTA orchestrator ↔ ota_node_status:** stage transitions (`canary→cohort→fleet→halted`) driven by simulated health-gate telemetry.

### 8.3 Canonical-contract conformance tests (NORMATIVE)

This band exists because a cross-document review found the same first-class objects defined three incompatible ways (`data-contracts.md` preamble). The contract is now single-source; **conformance tests make divergence a build failure.**

**The rule:** every producer and every connector must **round-trip** the `data-contracts.md` schemas — serialize to protobuf wire form, deserialize, and re-validate — with zero field loss and zero illegal values. This runs in CI for every service that emits or consumes a canonical object.

| Canonical object | Producers that must conform | Conformance assertions |
|---|---|---|
| `superheat.job.v1` (§4) | all connectors (`vast,runpod,salad,ionet,akash,direct,reserved,backlog`), yield router | `source ∈` full enum; `reliability_tier_required ∈ {probation,spot,standard,premium}`; durations in **seconds**; `net_price_usd_per_gpu_hr == gross·(1-platform_fee_frac)`; `gpu_model=="RTX4090"` exact token (reject `RTX_4090`, `RTX 4090`) |
| `superheat.telemetry.v1` (§5) | node agent (`node.md`), edge aggregator | `gpus` is **always an array** (length 1 for H1); `mem_used_mb` in MB; `net.path ∈ {direct,relay}`; carries **both** `thermal.mode` enum AND `thermal.heat_demand_w` numeric; `seq` monotonic |
| reliability score (§3) | scoring service | property test: output ∈ [0,100]; perfect node == 100; monotonic in each positive input |
| `superheat.usage.v1` (§8) | node agent (producer side), control plane (counter-sign) | `gpu_slot` mandatory; both `node_sig` and `cloud_sig` present and verify; never spans period boundary |
| thermal constants (§6) | thermal coordinator, registry | imports per-SKU `usable_thermal_kwh`; **fails if any `12.0` literal is hardcoded** |
| `fits()` predicate (§7) | scheduler, yield router, utilization engine | all three call the *same* implementation; a shared golden vector set produces identical verdicts across all three (the "each had a slightly different check" guard) |

**Golden corpus.** A versioned directory of canonical objects (valid + adversarial-invalid) lives in the repo. Any schema change bumps the `vN` tag (`data-contracts.md` §9 "changes here are breaking") and must update the crosswalk and the golden corpus in the same PR — enforced by a CI check that fails if `data-contracts.md` changed without a corpus diff.

### 8.4 End-to-end tier

Thin, expensive, few. Full path on real infrastructure: marketplace connector (sandbox/mocked external API) → yield router → scheduler → thermal coordinator → **HIL node** → usage record → ledger → tape view → reconciliation. This is the `platform.md` sequence diagram exercised against a real bench node. E2E is where we prove the wiring; the *behavioral* claims (SLO, utilization, money) are proven in §9–§10 and §17.

---

## 9. Fleet-scale simulation

The scheduler/router/thermal-coordinator claims are statistical properties of thousands of heterogeneous nodes. You cannot validate p95 placement latency, `<1%` idle-leak, or `60%×70%` utilization on a single bench. We build a **discrete-event simulator (DES)** that models a synthetic fleet and runs the *real* control-plane code (scheduler, thermal coordinator, yield router) against it. The control logic under test is the production binary; only the nodes, marketplaces, and network are simulated.

### 9.1 Architecture

```
   ┌──────────────────────────────────────────────────────────────┐
   │                      SIM HARNESS (DES core)                    │
   │  event queue · virtual clock · seeded RNG · metrics collector  │
   └───────────┬───────────────────────────────────┬──────────────┘
               │ telemetry frames                   │ placements / windows
               ▼                                     ▲
   ┌──────────────────────┐                ┌────────────────────────┐
   │  SYNTHETIC FLEET      │                │  CONTROL PLANE UNDER    │
   │  N nodes (model):     │                │  TEST (real code)       │
   │  - thermal sim (tank) │                │  - scheduler            │
   │  - climate/season     │◄──dispatch─────┤  - thermal coordinator  │
   │  - NAT/relay/link     │                │  - yield router         │
   │  - reliability events  │──telemetry────►│  - scoring              │
   └──────────────────────┘                └────────────┬───────────┘
   ┌──────────────────────┐                              │ jobs
   │  SYNTHETIC DEMAND     │──normalized jobs────────────┘
   │  marketplaces + back- │
   │  log + reserved + DR  │
   └──────────────────────┘
```

Each simulated node carries a **thermal twin** implementing the first-order energy balance of the utilization engine (`platform.md`) — `T_tank(t+Δt) = T_tank + (P_in − D − loss)·Δt/C_th` — parameterized by `data-contracts.md` §6 constants. Δt = 5 min for thermal; the event queue carries finer-grained job/network events.

### 9.2 Simulator I/O specification

**INPUTS (scenario file, versioned, seeded):**

| Input | Type | Drawn from / range | Models |
|---|---|---|---|
| `fleet_size` | int | 1k, 10k, 100k | scale tiers |
| `sku_mix` | dist | H1-4090 vs COMM-4090x8 | hardware heterogeneity (`platform.md`) |
| `region_mix` | dist over regions | us-east/mtn/… | sharding, data locality |
| `climate_profile[region]` | curve | HDD + DHW occupancy by month | thermal demand `D(t)` (utilization engine, `platform.md`) |
| `season` | enum/date | winter / shoulder / summer | seasonality regime (R5) |
| `tank_params[sku]` | C_th, T_min, T_max, ΔT_usable, η_capture | `data-contracts.md` §6 | tank-as-battery |
| `net_profile[node]` | direct \| CGNAT-relay; up/down mbps; rtt; loss | residential vs commercial dist (`connectivity-and-data.md`, R2) | NAT/relay behavior, link quality |
| `reliability_seed[node]` | base uptime, fault rate, preempt rate | tier distribution | drives scoring (§8.1) |
| `demand_trace` | per-source bids, fill prob, fee, payout latency, FX | marketplace rates (`platform.md`) | yield decisions |
| `backlog_depth` | queue model | always-available filler (`platform.md`) | utilization floor |
| `preemption_events` | rate / storm bursts | thermal close, yield preempt, DR shed | preemption mechanics (§9, §16) |
| `dr_events` | schedule | charge / shed | L4 DR (utilization engine, `platform.md`) |
| `fault_injection` | optional (§16) | XID, outage, bad OTA | chaos overlay |

**MEASURED OUTPUTS (per run + aggregate):**

| Metric | Definition | Asserted SLO | Owner doc |
|---|---|---|---|
| placement latency p50/p95/p99 | wall-time from job arrival to assignment for *available* capacity | **p95 < 2 s** (Part A S5) | `platform.md` |
| idle-window leak | fraction of usable thermal-window-minutes with no job for >5 min while backlog non-empty | **< 1%** (Part A S8) | `platform.md` |
| wrongful non-interruptible preemption | non-interruptible jobs evicted by an early-closing window ÷ premium placements | **< 0.5%** (Part A S7) | `platform.md` |
| utilization % | rented-and-running GPU-hours ÷ available GPU-hours | **≥ 60% commercial** (demand-gated) | utilization engine (`platform.md`) |
| thermal overlap % | run-hours whose heat is wanted-or-bankable ÷ run-hours | **≥ 70% commercial annual** | utilization engine (`platform.md`) |
| comfort violations | intervals where delivered temp < `T_min` attributable to compute | **0 (hard)** (Part A S1) | `platform.md` |
| window-prediction error | predicted vs actual `usable_seconds` | p50 within ±15% (Part A S18) | `platform.md` |
| yield-realized $/GPU-hr | mean net $ vs an oracle that picks the true max each hour | within X% of oracle | `platform.md` |
| concentration share | trailing-30d max single-marketplace revenue share | **≤ 60%** under guard | R6 |
| tier distribution drift | score/tier histogram over time | stable, no whipsaw (EWMA) | `platform.md` |

**PASS/FAIL.** A scenario passes iff every hard SLO holds (comfort=0, p95<2s, idle-leak<1%, concentration≤60%) and the soft targets (utilization, overlap, oracle gap) land within the scenario's declared band. Runs are **seeded and reproducible**; a failing seed is checked in as a regression scenario. Results are emitted as a machine-readable report keyed by scenario hash so CI can diff run-to-run regressions.

### 9.3 Required scenario matrix

At minimum these scenarios run nightly and gate the moat services:

1. **Steady commercial winter** — demand-rich, thermal loose; expect utilization demand-gated, overlap →100%, p95<2s.
2. **Residential summer DHW-only** — thermal *tight* (`E_day < GPU_max_daily`, utilization engine); validates the honest seasonal floor and that comfort is never violated when the tank saturates and the GPU must duty-cycle down.
3. **Mixed fleet, shoulder season** — realistic region/SKU/climate blend; the number we actually promise.
4. **Spot drought** — external demand collapses; assert backlog filler holds the floor and idle-leak stays <1%.
5. **Concentration pressure** — vast.ai >62% share; assert the guard steers marginal hours and share returns toward 60% without idling nodes.
6. **Preemption storm** — burst of thermal-close + yield-preempt; assert graceful-stop honored, no wrongful non-interruptible eviction, resume succeeds.
7. **Scale ramp** — 1k→10k→100k; assert placement p95 stays <2s (validates regional sharding / stateless-worker scaling, `platform.md`).

---

## 10. Control-system validation

The control loops must be validated not just on synthetic inputs but against **reality** — recorded and Phase-A telemetry — before any number is promised to a financier.

### 10.1 Replay testing

Phase-A nodes emit `superheat.telemetry.v1` to cold storage (Parquet, `platform.md`, 5-year retention). Replay harness:

- **Thermal-coordinator replay:** feed recorded `thermal.tank_temp_c`, `heat_demand_w`, and ambient into the coordinator; compare its predicted `usable_seconds` and `preheat_requested` decisions against what *actually* happened on the node. Assert window-prediction error p50 within ±15% (`platform.md`; Part A S18). This catches model drift before it reaches the fleet.
- **Reliability-score replay:** recompute scores over recorded heartbeat traces; assert `cloud_degraded` windows are correctly excluded (the false-outage bug, `data-contracts.md` §3 / R8; this is the Part A S16 exclusion); assert no node whipsaws tiers on a single bad day (EWMA behaves).
- **Scheduler shadow mode:** run the new scheduler against recorded job arrivals + node states and *log* placements without acting; diff against the production scheduler's historical decisions to quantify regressions before promotion.

### 10.2 Validating 60%×70% before it is promised

The economics base case is **60% utilization × 70% thermal overlap** (utilization engine, `platform.md`; `financial-and-roadmap.md` R5). This number underwrites the tape, so it must be *earned in simulation and confirmed in replay* before finance quotes it:

1. Run the §9.3 **mixed shoulder-season** and **per-region seasonal** scenarios using climate profiles calibrated from Phase-A warm-tier telemetry (`platform.md`).
2. Validate that the simulated utilization and overlap, weighted by the *regional seasonal* mix (not the winter peak — utilization engine, `platform.md`), meet the promised number with the filler + pre-heat + DR layers active.
3. Cross-check against replay: the realized overlap on Phase-A commercial nodes must not be materially below the simulated number. A gap is a model-honesty defect and blocks the underwriting input.
4. **`η_capture = 0.95` is a placeholder** (`data-contracts.md` §6); every duty-cycle conclusion is flagged conditional until the HIL rig (§12) and Phase-A nodes measure it. The sim runs a sensitivity sweep over `η_capture ∈ [0.85, 0.97]` so finance sees the band, not a point estimate.

The output is a **region-seasonal utilization curve** fed to underwriting — the explicit deliverable R5 demands ("no single flat 70% assumption").

---

## 11. Staging, canary environments

(See §13 for the release gate and §16 for chaos that run inside these environments.)

| Env | Composition | Purpose |
|---|---|---|
| **dev/CI** | unit + integration + conformance + sim-smoke | per-commit gate |
| **sim-nightly** | full §9 scenario matrix at 10k–100k | moat behavioral gate |
| **staging** | real cloud + HIL benches + chaos overlay | E2E + fault injection |
| **canary fleet** | 1% of real cohort, biased to low-reliability + idle/heating-only nodes, **never a non-interruptible reserved node** | first real-fleet exposure |
| **cohort/fleet** | staged waves | controlled rollout |

---

## 12. Hardware-in-the-loop (HIL)

Simulation validates the cloud control systems; HIL validates the thing that can physically hurt a customer. At least **one bench rig per region's hardware variant** runs the *exact* image and agent the field runs (`node.md`).

### 12.1 The bench rig

```
   ┌─────────────────────────────────────────────────────────────┐
   │  HIL BENCH                                                    │
   │  ┌───────────────┐   real GPU (4090)   ┌──────────────────┐   │
   │  │ node host      │◄──────────────────►│ thermal MCU       │   │
   │  │ immutable OS   │   UART/CAN/I²C      │ (real firmware)   │   │
   │  │ + superheat-   │◄──────────────────►│ heater_safe,      │   │
   │  │   agent        │                    │ comfort_constraint│   │
   │  └───────┬───────┘                    └─────────┬────────┘   │
   │          │ control loop (1 Hz)                  │            │
   │  ┌───────▼────────────────────────────────────▼─────────┐   │
   │  │  THERMAL EMULATOR  (real water tank OR validated      │   │
   │  │  electrical heat-load + simulated tank model)         │   │
   │  │  injects: sensor faults, over-temp, comfort draw,     │   │
   │  │  link loss, breaker-budget excursions                 │   │
   │  └───────────────────────────────────────────────────────┘   │
   │  ┌───────────────────────────────────────────────────────┐   │
   │  │  reverse-tunnel console + control-plane stub (cloud)  │   │
   │  └───────────────────────────────────────────────────────┘   │
   └─────────────────────────────────────────────────────────────┘
```

The thermal MCU runs **real firmware** because it is the local safety authority (`node.md`) — the `heater_safe=false` signal must be exercised on real silicon, not mocked.

### 12.2 Safety-critical test suite (must pass before ANY OTA)

These are blocking. A failure here halts the release pipeline.

| Test | Procedure | Pass criterion |
|---|---|---|
| **Fail-safe-to-heater** | kill `superheat-agent`; kill the whole OS; corrupt both USR slots | MCU keeps heating water on its own logic; customer never loses hot water (`node.md` MCU / recovery rung 4) |
| **`heater_safe=false` interlock** | MCU asserts `heater_safe=false` mid-job | control loop `HALT_GPU` within one tick (1 Hz); job preempted; GPU power-off observed (`node.md`) |
| **Thermal link loss** | sever MCU↔agent UART | `link_ok=false` → `HALT_GPU`; MCU's own watchdog forces conventional-heating-only; both sides fail safe |
| **Over-temp / breaker** | drive `tank_temp > TANK_TEMP_MAX`; drive `board_power > CIRCUIT_BUDGET_W` | HALT_GPU on over-temp; THROTTLE on circuit-budget; never trips breaker |
| **A/B health gate** | flash image with broken NVIDIA driver | gate fails, agent does **not** commit, auto-rollback to last-good slot; reports `rolled_back` (`node.md`) |
| **A/B hard rollback** | flash image that hangs before agent runs | bootloader auto-rollback after MAX(=3) attempts with no working agent |
| **GPU-broken → heater-safe** | image where GPU is dead but heat+phone-home work | passes to **heater-only safe mode**, does *not* rollback-loop (`node.md`) |
| **Recovery ladder** | inject faults at each rung 0→4 | each rung self-acts with no truck roll; HEATER-ONLY SAFE-MODE reachable from every state (R9) |
| **Attestation post-OTA** | commit new image | node re-attests; `image_digest ∈ allowed_release_set`; only attested node-hours billable (`node.md`) |
| **Watchdog** | hang the control loop | watchdog not petted → host reboots (rung 1) |

`η_capture` and `C_th` are **measured on this rig** with a real tank to replace the §6 placeholders, closing the open item flagged in `data-contracts.md` §6 and the utilization engine (`platform.md`). The HIL rig is the bench-side mirror of the field node that Part C, §1.4 installs and §5 services.

---

## 13. OTA gating, and the zero-hot-water release gate

### 13.1 OTA gating (canary → cohort → fleet) tied to the suite

The staged rollout of `platform.md` is **gated by this test suite**, not by time alone:

```
  CI green (unit+integration+conformance)
        └─► sim-nightly green (all hard SLOs, §9.2)
              └─► HIL safety suite green (§12.2 — BLOCKING)
                    └─► staging E2E + chaos green (§16)
                          └─► CANARY 1% (hold ≥6h)  ── health gate (platform.md) ──┐
                                └─► COHORT waves 5→25→50% (hold ≥2h/wave) ── gate ──┤
                                      └─► FLEET (capped N%/hr) ── gate ─────────────┘
                                            any breach ⇒ auto-halt <10min + auto-rollback + quarantine
```

Health-gate promotion criteria (from telemetry, `platform.md`) — vs a held-back control group: boot-success ≥99.5%, crash rate ≤1.5× baseline, GPU-fault/job-failure not elevated, and **thermal-safety faults = 0 (hard stop)**. (When this gate breaches in production, Part A Runbook 4.2 takes over.)

### 13.2 The zero-hot-water-incident release gate (checklist)

No image reaches canary, and no rollout advances to fleet, unless **every box is checked**. This is the operational form of R9 and the Part A **S1** SLO ("zero customers left without hot water due to an update"), stated once here and referenced by Part A's incident path.

- [ ] Unit tier green, including the §8.1 mandatory math (score==100, `fits()`, period-split, no `12.0` literal).
- [ ] Contract-conformance green — all producers round-trip `data-contracts.md` schemas; golden corpus updated if any `vN` bumped.
- [ ] sim-nightly: comfort violations = **0**; placement p95 < 2s; idle-leak < 1%; concentration ≤ 60%; utilization/overlap within scenario band.
- [ ] **HIL safety suite (§12.2) green** — fail-safe-to-heater, `heater_safe=false` interlock, link-loss, over-temp/breaker, A/B rollback (soft + hard), GPU-broken→heater-safe, recovery ladder all rungs, attestation post-OTA. **(Blocking.)**
- [ ] Chaos (§16): correlated outage, bad OTA, preemption storm all degrade gracefully; OTA auto-halt fires < 10 min on injected breach.
- [ ] Tape correctness (§17) green — golden + property + three-way reconciliation tests pass.
- [ ] Canary cohort filter excludes non-interruptible reserved nodes; biased to idle/heating-only + low-reliability.
- [ ] Rollback rehearsed: auto-rollback observed on the bench for this exact image digest.
- [ ] On-call + runbook ready; auto-halt and quarantine wired to the live rollout (Part A, §3–§4).
- [ ] Thermal-fault count attributable to this OTA across canary+cohort = **0** before fleet promotion.

A single failed box halts the release. The gate is deliberately conservative — canary→fleet for a clean release is allowed to take **< 14 days** (`platform.md`; Part A S15): safety over speed.

---

## 14. Tape / metering correctness tests

The tape **is** the product (`financial-and-roadmap.md` thesis; R8 "kills deals"). Metering correctness is therefore tested with the highest rigor in the pyramid: **property tests** (invariants must hold for all inputs) plus **golden tests** (known inputs → exact expected output, regression-locked).

### 14.1 Property tests (invariants over generated inputs)

For randomly generated job/usage streams, these must always hold:

- **Double-signing:** every `usage_record` on the tape has a valid `node_sig` AND `cloud_sig`; an unsigned, single-signed, replayed (duplicate `usage_id`), or tampered record is **rejected at counter-sign** and never enters `usage_record` (`data-contracts.md` §8).
- **Per-GPU, not per-node:** the tape aggregation joins `usage_record` to `gpu` on **both** `node_id` AND `gpu_slot`; for any multi-GPU node, total tape revenue == sum of per-GPU revenue (no Cartesian multiply — the explicit `data-contracts.md` §8 bug guard).
- **Period-boundary split:** no `usage_record` spans a billing-period boundary; a job crossing a month boundary produces exactly two adjacent records whose `gpu_seconds` sum to the original; monthly `utilization_pct` is exact.
- **Conservation:** `gross_usd == fee_usd + net_usd` per record (within rounding); `Σ net_usd` over a node/period equals the settlement basis.
- **Telemetry is never the tape:** assert no tape figure is derived from `superheat.telemetry.v1` (the lossy path) — only from signed usage records (the authoritative-data rule, `platform.md`; this is the two-budget separation from the governing principles).
- **Idempotent tape rebuild:** re-running the `securitization_tape` materialized view over the append-only `usage_record` table reproduces the identical tape (the "re-runnable view = audit guarantee", `platform.md`).

### 14.2 Golden tests (exact, regression-locked)

A curated fixture set of usage-record streams with hand-verified expected tape rows (GPU-hours, gross, realized $/GPU-hr, `utilization_pct`, reliability fields). Any change to the ledger or tape SQL that alters a golden output fails CI and demands explicit re-blessing. Includes the known edge cases: month-boundary job, multi-GPU commercial node, GPU module-swap mid-period (old serial retained for lineage — see Part C, §5.4), and a `probation`/unseasoned node correctly **excluded** from the tape.

### 14.3 Three-way reconciliation tests (R8)

The R8 mitigation is a **monthly three-way reconciliation**; we test it as a first-class harness against synthetic and replayed data (this is the harness behind Part A Runbook 4.5):

```
   signed usage records  ──┐
   (the metered truth)     │
                           ├─► reconcile ─► variance report
   marketplace settlement ─┤      │
   (platform.md)           │      ├─ matched & within tolerance → RECONCILED (settle)
                           │      ├─ matched, > tolerance        → FLAG variance (fee drift)
   bank cash receipts ─────┘      ├─ payout, no delivery record  → ORPHAN (investigate)
                                  └─ delivery, no payout past SLA → MISSING (arrears)
```

Test assertions:

- The ±tolerance applies to **signed usage records vs marketplace settlement**, NOT to lossy telemetry (the corrected claim, `data-contracts.md` §8; supersedes the old "reconcile telemetry within ±2%").
- Reconciliation drift > tolerance opens an audit ticket and **blocks settlement for the affected node** within one period (`platform.md`; Part A S11).
- Token-source FX (io.net/Akash) reconciles realized slippage against the router's decision-time `fx_haircut`; persistent bias feeds back (`platform.md`) — tested with synthetic FX traces.
- Orphan/missing/variance buckets are produced exactly for the injected anomalies; no false positives on clean data.

This harness is the Phase-B exit gate R8 names ("reconciliation tolerance threshold as a Phase-B exit gate") and produces the external-audit-ready variance report the warehouse close requires.

---

## 15. (reserved)

*Tape correctness lives in §14; chaos in §16. This number is intentionally retired to keep §16 chaos adjacent to §13 release gates in the reader's mind.*

---

## 16. Chaos / fault injection (ties to R9)

Beyond the bench, we inject correlated and adversarial faults into the simulator (§9) and the staging fleet (§11) to prove **graceful degradation** and the **recovery ladder**. This is the standing experiment R9 requires ("chaos/fault-injection on the recovery ladder"). The DR drills in Part A, §5 draw on this same fault-injection capability.

| Fault scenario | Injection point | Expected graceful behavior | Asserts |
|---|---|---|---|
| **Correlated cloud/aggregator outage** | sim + staging: drop regional control plane | nodes keep heating, keep running dispatched jobs, **buffer telemetry ≥2h** and usage records, reconcile on reconnect; `cloud_degraded` stamped so uptime is not penalized | `platform.md`; `data-contracts.md` §3 exclusion (Part A §3.3, S16, S19) |
| **Bad OTA** | push image failing health gate to canary cohort | per-node auto-rollback; fleet-wide `rolled_back` spike → orchestrator **auto-halt < 10 min**; affected nodes quarantined | `platform.md`; R9 (Part A Runbook 4.2, S13/S14) |
| **Link loss / CGNAT relay flap** | flip nodes direct→relay, raise loss | net_quality score degrades gracefully; data-heavy jobs steered off the node; no crash | `connectivity-and-data.md`, R2; scoring (§8.1) |
| **GPU XID fault** | inject `xid_errors` rising | fault_penalty lowers score; node sheds paid work; not a comfort event | `data-contracts.md` §3; `node.md` |
| **Preemption storm** | burst of thermal-close + yield-preempt across fleet | graceful-stop honored (`T_GRACE_S=30`); resume from durable PoP checkpoint; no wrongful non-interruptible eviction | `data-contracts.md` §7; utilization engine (`platform.md`) |
| **Scheduler thundering herd** | 100k simultaneous job arrivals | placement p95 stays <2s or degrades to backpressure, never double-books a GPU | `platform.md` (Part A S5) |
| **Tape poisoning attempt** | feed unsigned / replayed / period-spanning usage records | rejected at counter-sign; reconciliation flags; never enters tape | §14; R8 |

Chaos runs are scheduled (game-days) and a subset runs continuously in staging. Every new failure mode discovered becomes a seeded §9 regression scenario.

---

## 17. Tape reconciliation harness — cross-reference

The three-way reconciliation harness is specified in §14.3. It is the test-side counterpart of Part A Runbook 4.5 (Tape reconciliation breach) and the Phase-B exit gate for R8.

---

## 18. Ownership & phasing (testing)

| Test surface | Owned by | Ships in (roadmap phase) |
|---|---|---|
| Unit + contract conformance | each service team; conformance corpus by data-contracts owner | Phase A (from first node) |
| Integration + replay harness | control-plane / SRE | Phase A → B |
| Fleet-scale simulator | scheduler/controls team | Phase B (gates the moat) |
| HIL rig + safety suite | node-software + thermal-controls | **Phase A (invariant — R9, never a phase)** |
| Chaos / fault injection | SRE | Phase A (recovery ladder) → fleet-scale in B/C |
| OTA gating + release gate | SRE + control-plane | Phase A |
| Tape correctness + reconciliation | finance-systems + ledger eng | Phase A (v0) → Phase B (three-way, exit gate) |

The HIL safety suite and the zero-hot-water release gate are **invariants from the first node** (`financial-and-roadmap.md`, R9) — they are not deferred to a later phase. The simulator and three-way reconciliation are the behavioral and financial gates of Phase B; the scale and chaos breadth grow into Phase C.

---
---

# PART C — Field Operations, Installer & Customer Onboarding, and Node Recovery

---

**Scope of this part:** the entire field-service surface of the fleet — how a node gets certified-installed, identity-bootstrapped, serviced, GPU-swapped at year-5, and recovered when a homeowner stops being a willing host. This is the operational part that makes the rest of the system physically real: every node in `platform.md`'s registry got there through a truck, an installer, and a commissioning test described here. It is the *source* of the units Part A monitors and the field-side mirror of the HIL rig in Part B (§12).

**Expands:** the partner-led distribution model (`../market-research/Superheat AI Data Center.md` Slides 6, 7, 9) and the lifecycle states in `platform.md`.

---

## 19. Why field operations is a first-class system

Superheat is **partner-led**: local HVAC contractors, plumbers, and builders own customer acquisition, site assessment, installation, and first-line maintenance. **Superheat owns the platform** — the hardware standard, the software, compute demand, installer standards, training/certification, monitoring, and revenue settlement (`../market-research/…` Slide 6).

That division of labor creates a hard requirement this part exists to meet: a **stranger with a wrench** must be able to install a securitization-grade collateral asset, bootstrap its cryptographic identity, and hand it off to the control plane — **without ever touching the customer's router** (governing principle 1), without Superheat staff on-site, and without being able to forge or extract the node's signing key. The asset that results is the literal collateral in the tape (`platform.md`); a sloppy install is a sloppy bond.

Three field-ops realities follow from the system principles:

1. **The installer is semi-trusted, like the homeowner.** The certification program and the provisioning flow (§21) assume the installer is competent but not a Superheat employee and not a holder of fleet secrets. Identity bootstrapping is designed so a malicious or careless installer cannot mint a fake node or steal a key (`security-and-compliance.md`, threat T9).
2. **A truck roll is the most expensive thing we can do.** The recovery ladder (`node.md`) exists precisely so that almost nothing requires a visit. Field service (§22) is the fallback, not the default, and its cost feeds `financial-and-roadmap.md` (field-service/RMA OPEX line). This is the field-ops side of Part A §3.4's "maximize what the ladder fixes without a truck roll."
3. **The homeowner can walk away.** Unlike a data-center rack, the collateral lives in someone else's basement and depends on their continued willingness to host. Node recovery (§24) is the esoteric-ABS analog of auto-loan repossession and is the centerpiece risk this part adds to the register.

---

## 20. Installer certification & training

### 20.1 Why certification is load-bearing

The installer is performing three regulated/skilled trades at once on a single appliance: **electrical** (240 V branch circuit, breaker, possibly a sub-panel), **plumbing** (tank, heat-exchanger loop, dry-disconnect coolant couplers), and **node commissioning** (the Superheat-specific provisioning + attestation handshake). A failure in any one can (a) leave a customer without hot water — violating the fail-safe-to-heater invariant from the *outside*, (b) create a fire/scald/leak hazard, or (c) enroll a node that never attests and therefore never earns, polluting the tape. Certification is the quality gate on collateral origination.

### 20.2 Certification tiers

| Tier | Name | Scope | Prereqs | Recert |
|---|---|---|---|---|
| **T0** | Sales/Referral partner | Lead-gen + financing-application support only; **never** touches hardware | KYC of the business entity | Annual attestation |
| **T1** | Certified H1 Installer | Full H1 (1× 4090) home install: electrical tie-in, plumbing, commissioning, hand-off | Licensed electrician *or* plumber on staff + Superheat T1 course + 3 supervised installs | 24 mo + on any major hardware/SW rev |
| **T2** | Certified Commercial Installer | 8× 4090 commercial unit: 40 A / 208 V 3-phase, redundant PSU on dual circuits, cellular-failover provisioning, multi-zone thermal | T1 + commercial electrical license + 5 supervised commercial installs | 24 mo |
| **T3** | Certified Field-Service Tech | RMA, part swaps, **year-5 GPU module swap** (§23), dry-disconnect coolant work, on-site recovery escort (rung 3 escort) | T1/T2 + Superheat service course + coolant-handling cert | 12 mo (tighter — they open the safety domain) |

A given contractor business holds a tier; individual technicians hold a personal credential tied to their badge ID, which is recorded as `installer_id` on the `owner`/`node` records (`platform.md`) and stamped on every lifecycle event (`actor='installer:<badge>'`).

### 20.3 What a T1/T2 installer must be certified to do

- **Electrical:** verify branch-circuit capacity, confirm the unit shares (H1) or is provisioned (commercial) the correct circuit per `node.md`; confirm the **two-domain split** — heating element on its own control domain so a dead compute node still heats (governing principle 2). The installer must *never* be required to size compute power; the unit's `CIRCUIT_BUDGET_W` is SKU-derived (`node.md`).
- **Plumbing/thermal:** mate the tank, heat-exchanger loop, and **dry-disconnect coolant couplers** (`node.md`) without introducing air or leaks near electronics; populate the correct climate-variant sensor set (`node.md`).
- **Node commissioning:** run the guided commissioning app (§20.4, §21.4) end to end until the node reaches `attested` in the registry. The installer **does not** handle keys, edit router config, or open ports — the flow forbids it by construction (§21).
- **What they are explicitly NOT allowed to do:** open the GPU/safety domain (T3 only), extract or generate identity keys, override thermal interlocks, or modify the homeowner's network.

### 20.4 Training program & standards

- **Course:** online theory + hands-on lab on a bench unit + 3–5 supervised installs signed off by a T3 mentor. Covers the install workflow (§20.5), the provisioning flow (§21), the recovery ladder so they recognize what is *not* a truck roll, and the host-agreement/onboarding conversation (§25).
- **Standards pack:** a versioned install standard (torque specs, coupler procedure, circuit checklist, commissioning acceptance criteria). Tied to the hardware rev; a major rev forces recert.
- **Quality feedback loop:** every install's commissioning telemetry (first-boot attestation success, first-30-day reliability score, early RMA rate) is attributed back to the installer badge. Installers with elevated early-failure or early-RMA rates are flagged for re-training or suspension — this protects tape quality and feeds the **servicer-concentration** mitigation (`financial-and-roadmap.md` R7: a deep certified network is itself a backup-servicer asset, as Part A §6.2 notes).

### 20.5 Site assessment & install workflow

#### 20.5.1 Pre-install site-assessment checklist (gate before a truck is scheduled)

| Domain | Check | Pass criterion |
|---|---|---|
| **Electrical** | Branch circuit & panel capacity | H1: existing 240 V/30 A water-heater branch present and code-compliant. Commercial: 40 A single-phase or 208 V 3-phase available, or sub-panel feasible. |
| **Thermal load** | Annual heat demand / overlap | Estimated DHW (+ space-heat for cold-climate variant) sufficient that the climate-variant utilization curve clears the underwriting floor (`node.md`; `financial-and-roadmap.md` R5). Warm/DHW-only flagged as spot-tier site. |
| **Network** | Residential uplink | A working internet connection with *any* outbound path. **No** static IP, port-forward, or router admin needed (governing principle 1). Upload speed measured (scored, not gated — `node.md`). Commercial sites confirm cellular-failover coverage. |
| **Physical/plumbing** | Space, water, drainage, ambient | Tank footprint, water supply, drain/overflow, ambient heat-shed margin (warm-climate enclosure variant). |
| **Host eligibility** | KYC + agreement | Host-agreement signed, transfer-on-sale clause acknowledged, payout method on file, financing approved (§24.1, §25). |

A failed electrical/thermal/network check either re-scopes the SKU (e.g., warm-climate variant, spot-tier expectation) or disqualifies the site **before** dispatch — a failed install is the most expensive way to learn the site was wrong.

#### 20.5.2 Lifecycle mapping (this is where `provisioned → installed → attested → active` happen)

The install workflow is the physical realization of the registry lifecycle (`platform.md`):

```
[factory/warehouse]              [the truck roll]                      [first boot, no human]
 provisioned  ───────────────►  installed  ─────────────────────────►  attested  ──────────►  active
   │ node record created,          │ installer marks                     │ node self-attests       │ scheduler
   │ TPM key injected,             │ commissioning complete              │ (image+identity OK,     │ assigns
   │ enrollment token bound        │ in the app; lifecycle               │ §21.5) → control plane  │ paid work
   │ (§21.1)                       │ event actor='installer:<badge>'     │ flips state             │
```

- `provisioned`: created at manufacture/staging (§21.1) — *no installer involved yet*.
- `installed`: set by the installer in the commissioning app once physical + commissioning tests pass (§20.5.4). The installer can reach `installed` but **cannot** reach `attested`.
- `attested`: set by the **control plane**, only after the node cryptographically proves a genuine image + registered identity (§21.5, `node.md`). No human can fake this.
- `active`: set by the scheduler on first paid assignment; `enrolled_at` (tape origination date) is stamped here.

#### 20.5.3 The install workflow (numbered)

1. **Dispatch with a provisioned unit.** Warehouse ships a unit already in `provisioned` state with its TPM identity injected and a one-time enrollment token bound to *that* node record (§21.1). The installer never carries loose keys.
2. **Site re-verify.** On arrival, re-run the electrical/network spot-checks from §20.5.1 (panel breaker, live circuit, internet reachable).
3. **Plumbing/thermal install.** Mount the tank/heater, mate the heat-exchanger loop, connect dry-disconnect coolant couplers, install the climate-variant sensor set.
4. **Electrical tie-in.** Connect the heating-element domain and the compute domain on their respective circuits; confirm the heating element works **stand-alone** (compute powered off) — proves the fail-safe floor before compute is ever energized.
5. **Network connect (outbound-only).** Plug the node into the home router by Ethernet (Wi-Fi fallback). **Do not** open ports, set a static IP, or enter the router admin — the node dials out (§21.2). For commercial, insert the cellular SIM and confirm failover.
6. **Power on → first boot.** The node enters `PROVISIONING` (agent state machine, `node.md`): it dials out over the overlay, enrolls with its enrollment token, and obtains overlay credentials (§21).
7. **Commissioning test (automated, app-driven).** The commissioning app (running on the installer's phone/tablet, talking *only to the control plane*, never to the node directly) drives the acceptance suite (§20.5.4) and shows pass/fail.
8. **Attestation-at-first-boot.** The node runs the health gate (`node.md`) and answers the control plane's attestation challenge (§21.5). On pass, the control plane flips the node to `attested`.
9. **Installer marks `installed` / completes hand-off.** Once commissioning is green and attestation has cleared, the installer closes the work order; the lifecycle event records the installer badge.
10. **Customer hand-off & onboarding.** Walk the homeowner through expectations, payout setup, support channels (§25); leave the unit running as a normal water heater with compute enabled.
11. **First paid job → `active`.** The scheduler assigns the node, `enrolled_at` is stamped, and the node begins producing tape rows. The 30-day **seasoning** clock starts (`data-contracts.md` §3) — the node is not yet financeable collateral, but it is earning.

#### 20.5.4 Commissioning acceptance test (the green-light gate)

The commissioning app verifies, via the control plane (not by trusting the installer's word):

- **Heater-only proof:** the unit heated water with compute disabled (step 4) — the fail-safe floor is real.
- **Thermal link & sensors:** `thermal_iface` healthy, all climate-variant sensors reporting plausible values (`node.md` heater-critical checks).
- **Overlay & control-plane reachability:** the node established its outbound overlay connection and is delivering telemetry (`net check --control-plane` passes).
- **GPU smoke test:** `nvidia-smi` count matches SKU, CUDA smoke test passes (compute-optional checks). A GPU failure here does **not** fail commissioning fatally — the unit commissions as heater-only and the GPU is flagged for service, because the heater is the product floor (`node.md` `commit_degraded`).
- **Attestation:** the node reached `attested`.

Only when heater-only + thermal + overlay + attestation are green is the install accepted. (GPU-down is an accepted, flagged degraded state, not a failed install.)

---

## 21. Node provisioning & identity bootstrapping at install (critical)

This is the most security-sensitive part of field ops: a semi-trusted installer must bring a node's cryptographic identity online without holding keys, without touching the router, and in a way that lets the control plane prove the node is genuine before it ever earns.

### 21.1 Where the identity key comes from (and who injects it)

- **The signing key is born in the node's hardware secure element.** Each node carries a **TPM 2.0 / secure element** (on the BOM — `node.md`). The node identity key is **generated inside the TPM at manufacture/staging and never leaves it.** The installer never sees, generates, or transports a private key. (T9 mitigation, `security-and-compliance.md`.)
- **Injection happens at the factory/warehouse, not in the field.** During `provisioned`-state staging, Superheat registers the node's **public** key to its registry record (`node.attestation_pk`, `platform.md`) and binds a **one-time, short-lived enrollment token** to that node record. The private key stays sealed in the TPM (`node.md`).
- **Software-only fallback (lower assurance):** a node lacking a secure element falls back to a software-held key (`node.md` note). Such nodes are tracked as a lower assurance class and are **not** eligible for the `trusted_host` confidentiality class (`data-contracts.md` §2.2). Field-ops policy: secure-element-equipped is the standard; software-only is exception-only and flagged on the tape.

### 21.2 Enrolling into the registry + overlay without touching the customer's router

The entire point of the outbound-only design (governing principle 1) is that **the installer does nothing to the home network.** Concretely:

- The node **dials out** to the overlay coordination servers (WireGuard/headscale, `connectivity-and-data.md`) on first boot. No inbound port, no port-forward, no static IP, no router admin (`node.md` — "the installer must not have to touch the customer's router config").
- Behind CGNAT, the node falls back to relay automatically — still outbound-only. The installer is never asked to debug NAT.
- The commissioning app talks to the **control plane**, not to the node, so even the installer's device never needs a path *to* the node — eliminating "the installer opened a hole" as a class of error.

### 21.3 First-boot enrollment handshake

1. Node boots into `PROVISIONING`; agent reads its TPM-sealed key and the staged enrollment token from the `OEM` partition (`node.md`).
2. Agent dials out over the overlay to the enrollment endpoint.
3. Agent presents `{node_id, enrollment_token, hw_key_pub}` and proves possession of the private key (signs a server nonce).
4. Control plane verifies the token matches the staged node record and the `hw_key_pub` matches `node.attestation_pk`. Token is one-time and burns on use.
5. Control plane issues long-lived overlay credentials and confirms registry binding. Node is now reachable (outbound) and managed.

### 21.4 How a field tech does this safely

The tech's only tool is the **commissioning app**, which is a thin client of the control plane. It can: scan the unit's QR/serial to pull up the `provisioned` record, drive the acceptance suite, and display pass/fail. It **cannot**: read keys, set lifecycle to `attested`, edit network config, or talk to the node out-of-band. A malicious installer's worst case is a *failed* commissioning (the node won't attest), never a *forged* one — the cryptographic identity is the gate, not the installer's say-so.

### 21.5 Provisioning / attestation sequence diagram (attestation-at-first-boot)

```
 Installer App        Node Agent (TPM)          Overlay/Enroll        Control Plane          Registry/Tape
 (ctrl-plane client)  (node.md identity)         (connectivity-...)    (platform.md)          (platform.md)
      │ scan serial         │                          │                     │                       │
      │────────────────────────────── GetNode(provisioned) ───────────────► │── read node record ──►│
      │                     │ power on → PROVISIONING   │                     │                       │
      │                     │ read TPM key + token      │                     │                       │
      │                     │── dial OUT (no inbound) ─►│                     │                       │
      │                     │   {node_id, token,        │                     │                       │
      │                     │    hw_key_pub, sign(nonce)}│──── enroll ───────►│ verify token==staged, │
      │                     │                          │                     │ hw_key_pub==attestation_pk│
      │                     │◄──── overlay creds ───────│◄─── issue creds ───│  burn one-time token   │
      │                     │ run HEALTH GATE           │                     │                       │
      │                     │  heater-critical PASS      │                     │                       │
      │◄── commissioning suite results (via ctrl plane) ───────────────────── │                       │
      │                     │◄──────── attest challenge(nonce) ──────────────  │                       │
      │                     │ gather measurements:      │                     │                       │
      │                     │  image_digest, agent meas,│                     │                       │
      │                     │  secure-boot/dm-verity,   │                     │                       │
      │                     │  thermal-link present      │                     │                       │
      │                     │── signed_quote{nonce,meas}(TPM key) ──────────► │ verify digest∈allowed,│
      │                     │                          │                     │ sig chains to hw_key  │
      │                     │                          │                     │── lifecycle:          │
      │                     │                          │                     │   installed→ATTESTED ─►│ (eligible
      │ installer marks "installed" complete ───────────────────────────────►│   actor=installer     │  for paid
      │                     │                          │                     │                       │  work)
      │                     │  ... first paid job ...   │                     │── ACTIVE, stamp ─────►│ enrolled_at
      │                     │── superheat.usage.v1 (node_sig) ──────────────► │ counter-sign → tape   │ (tape origination)
```

The invariant: **only an attested image + registered identity node-hours ever enter the tape** (`node.md`). The installer can get a node to `installed`; only cryptography gets it to `attested`. (This is the same attestation invariant the HIL "Attestation post-OTA" test in Part B §12.2 exercises on the bench.)

---

## 22. Field service & RMA

The governing economics: a truck roll is the most expensive operation in the fleet, so the recovery ladder (`node.md`) resolves nearly everything remotely (this is the field-ops execution of Part A §3.4's action surface). Field service is the residual, budgeted as `financial-and-roadmap.md` field-service/RMA OPEX (~$160/unit-yr `[A]`, ~$0.0038/GPU-hr, ~linear with units, offset by installer-network density + the remote ladder).

### 22.1 Remotely fixable vs truck roll

| Symptom | First resort (remote) | Mechanism | Truck roll only if… |
|---|---|---|---|
| Bad image / boot loop | A/B auto-rollback | `node.md` | both slots bad **and** console can't re-image |
| Agent hung / kernel wedged | HW watchdog reboot | `node.md` rung 1 | repeated wedges after rollback |
| GPU XID errors / driver fault | OTA fix or heater-only degrade + flag | `node.md` `commit_degraded` | card hardware-dead (RMA) |
| Software config drift | Remote console over reverse tunnel | rung 3, `connectivity-and-data.md` | unrecoverable in software |
| Thermal-link flap | Self-diagnose; heater-only safe-mode | `node.md` | sensor/coupler hardware fault |
| Network down | Relay fallback / cellular (commercial) | `connectivity-and-data.md` | physical link cut / router gone |

If it can be fixed over the outbound overlay, it is **not** a truck roll. Rung 3 (remote console) is the last software resort and is outbound-only — never an inbound port on the home router.

### 22.2 RMA decision tree (when remote recovery is exhausted)

```
                         Node un-recovered after recovery ladder
                                        │
                          Is the HEATER still working?
                          ┌──────────────┴───────────────┐
                        YES                              NO
              (compute fault only)              (heater-critical fault)
                        │                                │
          Is fault isolated to the GPU?          URGENT truck roll — restore
          ┌─────────────┴──────────┐             hot water first (customer comfort
        YES                       NO              + host-agreement SLA, §24/§25).
   (GPU dead/degraded)     (host fault: CPU/      Heater MCU runs stand-alone in the
          │                 board/PSU/NVMe/RAM)   interim (node.md),
   Schedule a PART SWAP        │                  so this is degraded-not-dark.
   at next service window;     ▼
   node runs heater-only   host-failure question (node.md):
   meanwhile (degraded).   part-swappable in field?
   T3 tech swaps the       ┌──────────┴──────────┐
   GPU module (§23 proc). YES                    NO
                       Field part swap      WHOLE-NODE RMA:
                       by T3 tech;          de-install, ship to depot,
                       cheaper than RMA.    swap-in a spare unit.
                                            Tape: node → maintenance →
                                            (re-attest) active, or retired
                                            + substituted (§24 table).
```

> **Host-failure decision (open in `node.md`):** the GPU is designed as a field-replaceable module (§23), so **GPU failure = part swap**, not RMA. For host parts (CPU/board/PSU/NVMe/RAM) the trade-off is field-serviceability vs whole-node RMA. **Recommendation:** make **PSU and NVMe field-replaceable** (common, cheap, high-failure) but treat a **mainboard/CPU failure as a whole-node RMA** (swap in a spare unit, repair at depot) — opening the host deeply in a customer's basement is slow, error-prone, and risks the safety domain. This recommendation should be ratified into `node.md` and its cost folded into `financial-and-roadmap.md` field-service/RMA OPEX.

### 22.3 Spares strategy & SLAs across scattered sites

- **Regional spare pool.** Pre-positioned spare *whole units* and *part kits* (GPU modules, PSUs, NVMe, sensor sets) at regional depots near installer clusters. Whole-node RMA = swap-in-a-spare on-site, repair-at-depot — minimizes customer downtime.
- **SLA tiers track reliability tiers.** A `premium`/commercial node carrying contracted work gets a tighter on-site SLA (e.g., next-business-day heater restoration, defined-window compute restoration); a `spot` H1 gets best-effort. SLAs are written into the host agreement and the commercial site agreement (§24.1, `security-and-compliance.md`).
- **Heater-first SLA.** Regardless of tier, **loss of hot water is the only true emergency** — it triggers the urgent path in the RMA tree (§22.2) and is the same heater-first invariant Part A Runbook 4.3 dispatches against. Compute downtime degrades the tape (lower utilization) but never the customer relationship; hot-water loss threatens both (`financial-and-roadmap.md` R9).
- **Dispatch via the certified network.** RMAs are dispatched to local T3 techs (the same partner network), keeping truck-roll cost down and reinforcing the backup-servicer depth (R7; Part A §6).

### 22.4 Cost feedback to the financial model

Every field event (truck roll, part swap, whole-node RMA, spare consumption) is logged against the node and the installer badge, and aggregated into `financial-and-roadmap.md` field-service/RMA OPEX. Early-life RMA rate by installer feeds the certification quality loop (§20.4). Refresh-reserve draws (§23) hit the refresh-reserve line.

---

## 23. Year-5 GPU swap as a field operation

The financing model funds a **GPU-refresh reserve** (`financial-and-roadmap.md`, ~$1,000/unit-yr) for a year-5/6 module swap that re-ups the revenue curve for a second cycle. Executed across thousands of scattered sites, this is a field operation, not a depot operation — so the hardware is designed for it (`node.md`) and this section is the procedure.

### 23.1 Truck-roll logistics

- Swaps are **batched by region and installer cluster** to amortize travel, and **scheduled into thermal off-windows** (e.g., shoulder season for cold-climate sites) so compute downtime costs the least revenue.
- A T3-certified tech carries the new GPU module(s) and a return-RMA tray for the old cards (which retain residual resale/recycle value — falling card prices also cut swap CAPEX 40%+, `node.md`).
- The node is drained (current job checkpointed/migrated, `connectivity-and-data.md`) and put into `maintenance` in the registry **before** the tech arrives.

### 23.2 Certified-tech swap procedure

1. Confirm node is in `maintenance` and compute is drained; verify the unit is in heater-only safe-mode (still heating water for the customer throughout, governing principle 2).
2. **Dry-disconnect the coolant.** Use the quick-couplers so the loop is **not drained** — a card swap must not break the thermal loop (`node.md`). No coolant near electronics.
3. Power down the compute domain only (heater domain stays live; the customer never loses hot water).
4. Release the quick-release GPU carrier (captive thumbscrews / blind-mate power), remove old card(s), seat new module(s), re-mate coolant couplers, re-seat power.
5. Power up compute; the agent detects the new GPU (PCI ID, VBIOS, VRAM, NVML caps — `node.md`).
6. **Do not re-certify the safety domain.** A GPU swap must not touch the thermal interlock or heater control — the two domains stay separable by design (`node.md`).
7. Run the GPU smoke test + re-attestation; close the work order.

### 23.3 Version-aware re-registration handshake

The swapped card is a **new asset in a new generation** and the registry/tape must reflect it (`node.md`, `platform.md` registry):

1. Agent reports the new GPU to the registry; control plane writes a new `gpu` row in the same `slot_index` and sets `retired_at` on the old card's row — the `gpu_one_live_per_slot` unique index permits exactly this (`platform.md`).
2. Node **re-attests** (new image/inventory ⇒ attestation re-runs, `node.md`) before it can earn again.
3. Control plane re-registers the node at its new generation/SKU, tags it to the relevant **refresh-cycle tranche**, and resets the *card-reliability* seasoning baseline while keeping the site/network history (`node.md`).
4. The yield router reprices the node at the new card's tier (e.g., 4090→5090 lifts $0.40→$0.53-class demand).

### 23.4 Tape / collateral-generation change

This is a **securitization covenant event**, not just an ops event:

- The old card's `gpu` row is retained (`retired_at` set) for **tape lineage** — historical usage records still join correctly because `usage_record` references the exact `gpu_id` row (`platform.md`, `data-contracts.md` §8). The Cartesian-safe per-GPU join means the retired card's history and the new card's earnings never collide. (Part B §14.2 golden-tests this exact module-swap-mid-period case.)
- The tape shows the collateral generation cleanly: old cohort retired, new cohort begins, refresh reserve drawn down to fund the swap. Underwriters can see the second revenue cycle originate on the same physical site/network seasoning. (This is the `node.md` covenant requirement, met by the procedure here.)

---

## 24. Homeowner churn / node recovery (the collateral-risk centerpiece)

This is the field-ops surface that was **missing from the design and the risk register**, and it is the esoteric-ABS analog of **auto-loan repossession**: the collateral lives in a stranger's home and depends on their continued willingness to host. When a homeowner sells the house, unplugs the unit, revokes the network, or simply stops being a willing host, the node — *which is the securitization collateral* — stops producing tape, and unlike a car it cannot simply be towed.

### 24.1 Contractual remedies (the host agreement)

The first line of defense is legal, authored in `security-and-compliance.md` and referenced here as a field-ops dependency:

- **Host agreement, signed at onboarding (§25).** A multi-year **term** commitment to host the unit, keep it powered and networked, and provide reasonable service access, in exchange for hot water + compute payout share.
- **Transfer-on-sale clause.** On sale of the property, the agreement **runs with the appliance**: the seller must either (a) assign the host agreement to the buyer (preferred — the unit stays put, a new `owner` record is created, payout reassigned), or (b) trigger a Superheat-managed **de-install + re-siting** (§24.3). This is the single most important clause — it converts a house sale from a collateral loss into an ownership transfer.
- **Lien / ownership clarity.** Superheat (or the SPV) owns the compute hardware; the homeowner owns the heating service. Clear title + UCC fixture-filing considerations so the collateral is perfected and not lost as a fixture in a property sale or foreclosure (`security-and-compliance.md` — flagged as a hard dependency).
- **KYC of the host.** Direct hosts are KYC'd at onboarding (`security-and-compliance.md`, T7), giving a real counterparty for the agreement and for recovery.

### 24.2 Economic remedies

- **Payout as the carrot.** The host earns only while the node is hosted, powered, networked, and attested. Going dark forfeits payout — a structural incentive to stay a willing host.
- **Early-termination / removal economics.** The host agreement defines what happens if a host removes the unit before term: e.g., the host loses the subsidized heating economics and may owe a removal/de-install fee or the residual financed balance, mirroring how POS-financed solar handles early removal. (Exact terms: `security-and-compliance.md`.)
- **Re-siting beats write-off.** Because the unit is standardized, recovered hardware has high re-deployment value — the economically rational remedy is usually **recover-and-re-site** (§24.3), not write-down, which protects the pool.

### 24.3 What happens to a "lost" node — registry/tape state transition & collateral treatment

A node that stops hosting moves through explicit registry states (extending `platform.md` lifecycle) and is treated explicitly in the tape:

```
active ──goes dark──► degraded ──confirmed-removed/revoked──► quarantined ──┬─► maintenance (recover) ─► re-sited ─► active (re-attest, new owner/site)
                                                                            └─► retired (write-down) ─► collateral SUBSTITUTION in the pool
```

- A node confirmed lost is taken out of the **earning** pool immediately (stops polluting utilization/uptime metrics for its tranche).
- For the tape: a lost node is either **recovered + re-sited** (preferred — same hardware, new site, re-seasons) or **retired + substituted** — the SPV substitutes an eligible seasoned node from the substitution pool so pool size/quality is maintained, exactly as esoteric ABS deals handle defaulted/removed collateral. A net loss after recovery value is a **collateral write-down** charged against residual/credit enhancement (this is the repo-loss analog).
- **Concentration of churn** (e.g., a region or installer cohort with elevated churn) is a covenant signal, reported on the tape like any other collateral-quality trend.

### 24.4 Node re-siting / redeployment

A recovered unit (from a house sale, removal, or whole-node RMA spare-swap) is refurbished/validated at a depot, returned to `provisioned`, and re-installed at a new site via the normal install workflow (§20.5–§21) — **including a fresh first-boot attestation and a fresh seasoning clock for the site/network** (the card history may carry, but the site reliability re-seasons, mirroring §23.3). Standardization is what makes re-siting cheap and is why recover-and-re-site is the default remedy.

### 24.5 Detection — dark vs removed

The hard problem: a node that has **gone dark** (transient outage, ISP down, breaker tripped, customer on vacation) must be distinguished from one that has been **removed/revoked** (host churn), because the remedies differ entirely. This is the field-ops sibling of Part A §3.3's node-down-vs-cloud-down correlation test — both ask "is this a real loss or a false alarm?" Detection blends telemetry signals (`data-contracts.md` §5) the node already emits:

| Signal pattern | Likely cause | Action |
|---|---|---|
| Heartbeat gap, but `cloud_degraded` set (correlated across many nodes) | Superheat/ISP-regional outage | Excluded from uptime (`data-contracts.md` §3; Part A S16); **no action** — not the node's fault |
| Brief heartbeat gap, node returns, thermal sensors resume | Transient (power/network blip, vacation) | Watch; no recovery action |
| Heartbeat gap, then node returns on a **new network path/relay** but same TPM identity attests | Router/ISP change, possibly a move | Re-bind network; confirm site via host; possible re-site signal |
| **Clean shutdown** signal then prolonged silence; last telemetry showed unplug-pattern (power cut while heater also stops) | Deliberate unplug / removal | Open host-contact workflow → degraded → quarantine if confirmed |
| Node attests from a **wildly different geo / network** | Possible move or theft/relocation | Flag; verify against host agreement; treat as removal/transfer |
| Prolonged dark **past the host-agreement grace window**, host unresponsive | Host churn (sold/revoked) | Escalate to recovery: de-install or re-site (§24.3) |

The TPM identity is what makes this tractable: a node that reappears proves it's the *same* asset (genuine re-attestation), so "moved" vs "replaced" vs "gone" is cryptographically distinguishable, not guessed.

### 24.6 New named risk for the register

This part recommends adding to `financial-and-roadmap.md` risk register:

> **R12 — Homeowner churn / node recovery (collateral repossession analog).** The collateral lives in a third-party home and depends on continued willing hosting. House sale, unplug, network revocation, or an unwilling host removes earning collateral and, unlike a car, it cannot be towed. **Impact:** High (direct collateral loss / pool quality). **Likelihood:** Med (consumer behavior over a multi-year term, esp. on house sales). **Owner-area:** `operations.md` Part C §24, `security-and-compliance.md`, `platform.md` (lifecycle states). **Mitigation:** host agreement with **transfer-on-sale** clause; lien/fixture-filing perfection; payout-as-incentive; dark-vs-removed detection (§24.5); recover-and-re-site default; collateral **substitution** in the pool; churn-concentration reporting on the tape. **Resolving decision/experiment:** finalize the host agreement + transfer-on-sale + lien strategy with counsel before warehouse close; pilot a full recover-and-re-site cycle (detection → de-install → re-attest at new site) and measure recovery value vs write-down; set a pool churn-rate covenant.

---

## 25. Customer / homeowner onboarding & support

### 25.1 Enrollment

- **Lead → assessment → financing → agreement.** The installer (or T0 referral partner) brings the lead; the site assessment (§20.5.1) qualifies it; POS financing is applied for (solar-style, `../market-research/…` Slide 7); the **host agreement** (term, transfer-on-sale, access, payout) is signed (§24.1) and the host is KYC'd.
- **Owner record created.** An `owner` row (`platform.md`) is created with payout method (tokenized, never raw bank data) and tied to the `installer_id`.

### 25.2 Setting expectations (noise / heat / comfort)

The onboarding conversation (scripted in the training standard, §20.4) sets honest expectations:

- **Hot water is guaranteed and comes first** — comfort/safety constraints are inviolable; compute never compromises hot water (governing principle 2; `platform.md`). In the worst case the unit is just a water heater (`node.md`).
- **Heat & comfort:** the unit captures GPU heat into the tank; cold-climate variants tie into space heating. Warm-climate/summer = DHW-only, less compute (set expectations on payout seasonality, `node.md`).
- **Noise/footprint:** an H1 is a quiet appliance (heat goes to water, not a loud fan to the room — near-1.0 PUE). Set realistic expectations on any fan/pump noise and enclosure heat-shed.
- **Privacy & their network:** the renter workloads are isolated from the home LAN (`security-and-compliance.md`) — the homeowner's devices are never reachable by tenants.

### 25.3 Payout setup

- Compute-income share is settled per the four-party split (homeowner / installer / lender / Superheat — `../market-research/…` Slide 9) via the metering ledger and `settlement` table (`platform.md`). The homeowner sees payouts that offset their financed monthly payment (Slide 8).

### 25.4 Support channels

- **Tier-1: app/portal + chatbot** for status ("is it earning?", payout history, "is my hot water OK?").
- **Tier-2: Superheat platform support** for software/account/payout issues (remote — most issues are remote-fixable, §22).
- **Tier-3: local certified installer/T3 tech** for anything physical (the partner owns the local relationship — `../market-research/…` Slide 9). Heater-down is the one emergency path (§22.3).
- **Self-healing first:** most faults are resolved by the recovery ladder before the customer ever notices (`node.md`) — the best support call is the one that never happens because the unit fell back to heater-only and self-recovered.

---

## 26. Open items (field-ops-specific)

- **`security-and-compliance.md` owns the contractual layer.** The host agreement, transfer-on-sale clause, lien/UCC-fixture strategy, KYC, and removal economics (§24.1, §24.2) are hard dependencies on that doc — drafting the host agreement + transfer-on-sale + lien strategy with counsel is the gating action before warehouse close.
- **Host-failure RMA policy** (`node.md`): ratify the §22.2 recommendation (PSU/NVMe field-replaceable; mainboard/CPU = whole-node RMA).
- **Spares depot density vs SLA** — how many regional depots, how deep the spare pool, to hit per-tier SLAs at fleet scale without over-stocking. Feeds `financial-and-roadmap.md` field-service/RMA OPEX.
- **Dark-vs-removed detection thresholds** (§24.5): the grace windows and geo/network-change heuristics need field tuning against real residential behavior (vacations, ISP churn) to avoid false-churn escalations.
- **Re-siting yield** — empirical recovery value vs write-down on recovered units, to set the R12 covenant and the substitution-pool sizing.
- **Installer-quality model** — the early-failure/early-RMA attribution loop (§20.4) needs enough volume to be statistically actionable; until then, suspension thresholds are judgment calls.

---
---

*Cross-references (other docs):* `data-contracts.md` (§2 tier, §3 reliability score, §4 job, §5 telemetry frame, §6 thermal constants, §7 `fits()`, §8 signed usage record — all NORMATIVE; the SLI/tape/test sources); `system-design-overview.md` (the architectural overview these operations realize); `platform.md` (control-plane services, per-service SLOs consolidated in Part A §1, registry/lifecycle states, OTA, scheduler, thermal coordinator, yield router, tape, sequence diagram, scale/sharding); `node.md` (A/B rollback + health gate, control loop + state machine, recovery ladder, MCU interface, identity/attestation, image CI, TPM BOM, electrical/climate variants, year-5 swap, host-failure RMA); `connectivity-and-data.md` (outbound-only overlay, reverse-tunnel console, relay/TURN, checkpoint/migration); `security-and-compliance.md` (three-party trust, T7/T9 KYC/attestation, host agreement, transfer-on-sale, lien/fixture perfection); `financial-and-roadmap.md` (field-service/RMA + refresh-reserve OPEX lines, R5 seasonality, R6/R7 concentration & backup servicer, R8 tape integrity, R9 fail-safe-to-heater, and the new R12 homeowner churn).
*Internal cross-references:* Part A SLO catalog (§1) ⇄ Part B assertions (§9.2, §13, §17); Part A correlated-outage handling (§3.3) ⇄ Part C dark-vs-removed detection (§24.5); Part A truck-roll action surface (§3.4) ⇄ Part C field service & RMA (§22); Part A backup-servicer escrow (§6) ⇄ Part C certified network (§20); Part B HIL attestation (§12.2) ⇄ Part C provisioning/attestation (§21).
*Source consistency:* `../market-research/Superheat AI Data Center.md` (Slides 6, 7, 8, 9 — partner-led distribution, POS financing, four-party split).
