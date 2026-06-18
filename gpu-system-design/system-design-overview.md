# Superheat — GPU Compute Network System Design (Overview)

**Status:** Draft v0.1 · **Date:** June 2026 · **Audience:** Engineering + investors/partners
**Scope:** The operational/technical system that runs Superheat's distributed, heater-embedded GPU fleet and keeps it rented, utilized, and easy for renters to use.

This document is grounded in the business and market work in `../market-research/` and should stay numerically consistent with it. It is the *overview* and entry point: it carries the design principles, the architecture, the six founding questions, and — folded in at the end — the end-to-end walkthroughs. Each layer expands into one of the eight deep-dive docs listed in §0.

---

## 0. How this maps to the six founding questions

| # | Question (original) | Section | Deep-dive doc |
|---|---|---|---|
| Q1 | Remote operations & maintenance of scattered single GPU nodes | §4, §5, §6 | [`node.md`](node.md), [`connectivity-and-data.md`](connectivity-and-data.md), [`platform.md`](platform.md) |
| Q2 | What OS / software stack a single GPU node needs | §3, §4 | [`node.md`](node.md) |
| Q3 | RTX 4090 node config that satisfies the most renters at lowest cost | §3 | [`node.md`](node.md) |
| Q4 | Auto-dispatch compute + integrate with vast.ai-style marketplaces | §7 | [`platform.md`](platform.md) |
| Q5 | Build our own network to drive utilization toward zero idle | §7, §8 | [`platform.md`](platform.md) |
| Q6 | Efficient orchestration + smooth rental & data I/O across everywhere | §5, §6, §8, §9 | [`connectivity-and-data.md`](connectivity-and-data.md), [`platform.md`](platform.md) |

> **Q4 and Q5 are not a fork.** The chosen strategy is **hybrid**: attach to existing marketplaces *first* for instant utilization and revenue, build our own orchestration + direct/enterprise demand *in parallel*, and route every node-hour to whichever source pays most at that moment. See §7.

### The consolidated doc set (8 docs)

The design has been consolidated to these eight documents:

1. [`system-design-overview.md`](system-design-overview.md) — **this doc**: design principles, architecture, the six founding questions, and the end-to-end walkthroughs.
2. [`data-contracts.md`](data-contracts.md) — NORMATIVE canonical schemas: job, telemetry, usage record, `fits()`, tiers, thermal constants. The owning doc for every shared name, unit, and rule.
3. [`node.md`](node.md) — the GPU node: hardware (SKU, thermal coupling, year-5 swap) plus software/OS (immutable Linux, the agent, isolation, attestation).
4. [`connectivity-and-data.md`](connectivity-and-data.md) — the outbound-only overlay network (NAT/relay) plus data I/O (regional cache, checkpoint/resume, bandwidth-aware placement).
5. [`platform.md`](platform.md) — the cloud platform: control plane (registry, scheduler, thermal coordinator, reliability, ledger/tape), demand & yield routing, utilization engine, and the renter API.
6. [`security-and-compliance.md`](security-and-compliance.md) — security/trust (isolation, no-TEE honesty, attestation) plus legal/regulatory/compliance.
7. [`financial-and-roadmap.md`](financial-and-roadmap.md) — CAPEX + OPEX + waterfall + build roadmap + risk register.
8. [`operations.md`](operations.md) — observability/SLO/incident + testing/simulation + field ops & onboarding.

---

## 1. Executive summary & design principles

Superheat's fleet is unlike a data center: the GPUs live inside water heaters, in homes and commercial buildings, behind ordinary residential internet, and they can only run compute when their heat is wanted or can be stored. The entire architecture follows from five non-negotiable realities:

1. **Outbound-only.** Nodes sit behind residential NAT/CGNAT with dynamic IPs and asymmetric (slow-upload) consumer links. Nothing can rely on inbound ports. Every connection — control, console, renter access, data — is initiated *from* the node *outward*.
2. **Fail safe to a water heater.** Physical access to a node is rare and expensive. No update, crash, or compute fault may ever brick the unit or leave a customer without hot water. Heating is the floor; compute is the upside.
3. **Compute is thermally coupled.** A GPU should run when its heat is useful, *or* when the tank (a thermal battery) can absorb the heat for later. Scheduling is a joint compute-and-heat optimization, not pure compute.
4. **Measure, then price, reliability.** Nodes are heterogeneous and unreliable by data-center standards. We continuously score each node's uptime/network/thermal availability and route premium work to reliable nodes, spot/interruptible work to the rest.
5. **Hybrid demand.** Never depend on one buyer. Marketplaces give instant fill; our own direct/enterprise demand and an interruptible batch backlog give margin and a utilization floor.

**Design goal in one line:** turn thousands of unreliable, NAT'd, heat-constrained single-GPU boxes into one standardized, sellable, near-zero-idle compute fleet — with a metered revenue ledger clean enough to securitize.

---

## 2. System architecture overview

Five layers, bottom to top. Each renter request and each owner payout flows through all five.

```
                          ┌─────────────────────────────────────────────┐
                          │                  RENTERS                      │
                          │  (marketplaces, direct/enterprise, own API)   │
                          └───────────────────────┬─────────────────────┘
                                                   │
   ┌───────────────────────────────────────────────────────────────────────┐
   │  (5) DEMAND LAYER   marketplace connectors · own marketplace/API ·       │  §7
   │                     reserved contracts · interruptible batch backlog ·   │  platform.md
   │                     YIELD ROUTER (pick highest $/GPU-hr per node-hour)   │
   └───────────────────────────────┬───────────────────────────────────────┘
                                    │ normalized internal "job" model
   ┌───────────────────────────────────────────────────────────────────────┐
   │  (4) CONTROL PLANE (cloud)   registry/inventory · telemetry · OTA ·      │  §6
   │      scheduler · THERMAL COORDINATOR · reliability scoring ·             │  platform.md
   │      METERING + BILLING LEDGER ──────────► securitization tape           │
   └───────────────────────────────┬───────────────────────────────────────┘
                                    │ outbound-only control channel
   ┌───────────────────────────────────────────────────────────────────────┐
   │  (3) CONNECTIVITY   WireGuard overlay mesh (outbound-only) + relay/TURN  │  §5
   │                     fallback for CGNAT · carries control, console, data  │  connectivity-and-data.md
   └───────────────────────────────┬───────────────────────────────────────┘
                                    │
   ┌───────────────────────────────────────────────────────────────────────┐
   │  (2) NODE SOFTWARE   immutable Linux + A/B updates · NODE AGENT ·        │  §4
   │                      container/microVM tenant isolation · watchdog       │  node.md
   └───────────────────────────────┬───────────────────────────────────────┘
                                    │
   ┌───────────────────────────────────────────────────────────────────────┐
   │  (1) NODE HARDWARE   RTX 4090 SKU (home 1× / commercial 8×) · thermal    │  §3
   │                      coupling to tank · NIC + cellular failover          │  node.md
   └───────────────────────────────────────────────────────────────────────┘

         DATA I/O (§9) runs alongside (5)→(1): regional cache · pre-staging ·
                       checkpoint/resume · bandwidth-aware placement (connectivity-and-data.md)
```

The rest of the document details each layer.

---

## 3. The GPU node — hardware (Q2 config + Q3 cost & coverage)

### 3.1 Why RTX 4090

The 4090 is deliberately the fleet's workhorse, and the market data backs it (`../market-research/GPU_Pricing_Research_4090_H100.md`):

- It is the **cheapest training-class card per hour** (~$0.44 median, ~$0.40 typical, spot floor ~$0.15–0.25) and has been **price-stable near its floor** rather than crashing like flagship cards — ideal for a long-lived, financeable asset.
- It covers the **largest demand segment**: LLM fine-tuning and inference of mid-size models, diffusion/image/video generation, 3D rendering, and general batch ML. Most rental demand on Vast.ai-class marketplaces is exactly this tier.
- Its successor (5090) sits at only a ~35–40% premium, so the per-generation rental step-down is mild (~−20%/yr at constant tier), which protects unit economics over the 5–6-year revenue life.

**Strategy: standardize on one cost-optimized 4090 SKU**, refurbished cards (~$1,800 each per the economics memo), and add multi-GPU commercial nodes for jobs that need 2–8 GPUs. Standardization is what makes the fleet manufacturable, serviceable, and securitizable.

### 3.2 The "don't starve the GPU" spec floor

To satisfy the broadest set of renters *cheaply*, the rest of the box must be just enough that the 4090 is never the idle component for common workloads — and no more. Per-GPU minimums:

| Component | Per-GPU target | Why |
|---|---|---|
| CPU | ≥ 4–6 physical cores (≥ ~1 fast core/GPU + headroom) | Data loading / preprocessing for fine-tune & inference pipelines |
| System RAM | 32 GB/GPU (16 GB floor) | Many renter images assume ≥ RAM ≈ 1.5× VRAM (24 GB) |
| NVMe scratch | ≥ 1 TB fast NVMe/GPU | Model + dataset staging; checkpoint I/O (§9) |
| PCIe | Gen4 ×8–×16 per GPU | Avoid host↔GPU transfer bottleneck on multi-GPU nodes |
| Upload bandwidth | Best available residential; measured & scored | Result/checkpoint egress is the consumer-link bottleneck (§9) |

Going below these caps the renter workloads the node can accept and depresses its $/GPU-hr; going far above wastes CAPEX that the heat-offset economics don't need. This is the cost/coverage sweet spot Q3 asks for. Full BOM and the hardware design live in [`node.md`](node.md).

### 3.3 Two standard form factors

- **H1 home unit:** 1× RTX 4090, 3.3 kW hybrid water-heating system. Single-tenant per node.
- **Commercial unit:** multi-GPU chassis sharing one host. Can run multi-GPU jobs or be sliced into independent single-GPU rentals.

Reference economics (`../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md`): the memo's **$18K** fully-loaded commercial CAPEX is **likely understated to ~$20–22K** once a real 8-GPU compute host is priced — see the CAPEX reconciliation in [`financial-and-roadmap.md`](financial-and-roadmap.md), which restates payback to ~21–22 mo, IRR to ~33–37%, and stress DSCR to ~1.7×. Net revenue ~$11.7K/unit-yr at 60% utilization × 70% thermal overlap and a 25% platform fee (commercial; optimistic for residential/warm-climate — see the utilization engine in [`platform.md`](platform.md)). Near-1.0 PUE — the heat is the product, so there's no cooling overhead that a normal data center pays.

> **⚠ Spec conflict to resolve.** The pitch (`../market-research/Superheat AI Data Center.md`) describes the commercial unit as **12 GPUs / 33 kW**, while the economics memo models **8× RTX 4090 @ 412 W ≈ 3.3 kW (H1-class)**. The doc that drives BOM, thermal design, and the securitization tape must pick one definition. **Recommendation:** standardize the commercial SKU at **8× 4090** (matches the underwritten economics) and treat "12 GPU / 33 kW" as a larger optional configuration, *or* re-underwrite at 12. This must be decided before hardware freeze.

### 3.4 Thermal & power coupling (interface, not mechanical design)

Each 4090 draws ~412–450 W. Its heat is captured (water block / heat exchanger) into the tank, which acts as a **thermal battery** — the single most important fact for utilization (§8). Mechanical/plumbing design is out of scope here; the system design only requires that the node agent can **read tank temperature / heat demand and control GPU power state** through a thermal controller (§4).

---

## 4. The GPU node — software & OS (Q2)

### 4.1 OS: immutable / atomic Linux with A/B updates

Because a node may be physically unreachable for months, the OS must be **self-healing**, not hand-administered.

**Recommendation:** an immutable/atomic Linux base (e.g., **Ubuntu Core**, **Flatcar**, or a **Talos**-style appliance OS), with:

- **A/B partition updates + automatic rollback.** A bad update boots into the known-good partition instead of bricking. This is the single most important operational choice for an unreachable fleet.
- **NVIDIA driver + Container Toolkit** baked into the image; tenant workloads never touch the host driver directly.
- **Minimal attack surface** — read-only root, no general-purpose package management on the node, signed images only.

This trades the convenience of "just SSH in and apt install" for the reliability a physically-inaccessible fleet demands. It is the same reasoning behind ChromeOS/Tesla-style appliance updates.

### 4.2 The node agent (control daemon)

A single signed daemon owns the node. Responsibilities:

- **Heartbeat & telemetry:** power draw, GPU temp/util, tank temp & heat demand, uptime, network quality → control plane (§6).
- **OTA orchestration:** pulls staged image updates, applies A/B, reports result.
- **Container/microVM lifecycle:** launches tenant workloads in isolated containers (and microVMs for stronger isolation where supported).
- **Thermal & power policy enforcement:** starts/stops/throttles GPU work according to the thermal coordinator's plan and hard local safety limits — *local safety always wins over remote scheduling*.
- **Local job cache:** caches common base images, models, datasets (§9).
- **Hardware watchdog:** auto-reboots/recovers a hung node.

Full agent design, state machine, and attestation flow live in [`node.md`](node.md).

### 4.3 Isolation & recovery

- Tenant workloads run with **no access to the thermal controller, the host OS, or the home LAN.** A strict **egress firewall** limits what a renter container can reach (anti-abuse, §10).
- **Recovery ladder:** hardware watchdog → A/B auto-rollback → remote console over the reverse tunnel (§5) → last resort, the unit **falls back to operating as an ordinary water heater** with compute disabled, so the customer is never without hot water while we triage.

---

## 5. Connectivity layer — solving NAT (Q1 remote ops + Q6 reachability)

Residential nodes cannot accept inbound connections. The solution is an **outbound-only overlay network**.

- **WireGuard-based overlay mesh.** Each node dials *out* to coordination servers and joins a private mesh. Options: **Tailscale / self-hosted headscale**, or **Nebula**. The control plane, operators, and (mediated) renters then reach any node over the mesh with **zero inbound ports opened on the home router**.
- **Relay / TURN fallback.** On CGNAT, direct peer-to-peer hole-punching often fails; traffic falls back to a relay. Relay bandwidth is a real cost line — budget it and prefer direct paths when possible (this interacts with data I/O, §9).
- **One channel, many uses.** The same overlay carries control/telemetry, OTA, remote console/diagnostics, and renter session traffic — all authenticated, all encrypted, all node-initiated.

This single layer is what makes Q1 (remote ops of scattered nodes) and the reachability half of Q6 tractable at all. The overlay and relay design live in [`connectivity-and-data.md`](connectivity-and-data.md).

---

## 6. Control plane / cloud (Q1 ops + Q5/Q6 orchestration)

The cloud brain that turns scattered boxes into one fleet. Core services:

- **Registry & inventory.** Every node, its SKU, GPUs, location/region, owner, financing/tranche tag, current state.
- **Telemetry & observability.** Time-series of power, temp, GPU util, uptime, network quality. Drives dashboards, alerts, and scoring.
- **OTA update orchestrator.** Staged rollouts: canary → cohort → fleet, with automatic halt + rollback on regression. Never push to 100% at once.
- **Reliability/uptime scoring.** A continuously updated score per node (uptime %, thermal availability, network quality). Feeds the scheduler and the yield router; also feeds the securitization tape (seasoned, high-reliability pools are what get financed).
- **Scheduler / orchestrator.** Matches demand (§7) to nodes subject to **thermal availability + network quality + reliability score + interruptibility**.
- **Thermal coordinator.** Per-site model treating the **tank as a battery**: knows current temp, heat-demand forecast, comfort constraints, and storage headroom; tells the scheduler when each node can run compute and for how long, and does pre-heat scheduling (§8). Hard comfort/safety constraints are inviolable.
- **Metering & billing ledger.** Authoritative record of GPU-hours delivered, $/GPU-hr realized, owner settlement, platform take. This ledger **is the securitization tape** described in `../market-research/Superheat_Compute-Network_Debt-Engine_Onepager.md` — so it must be accurate, auditable, and per-unit from day one.

**Pragmatic building blocks (not prescriptive):** managed Postgres (e.g., Supabase) for registry + ledger; a purpose-built time-series store for telemetry; an established fleet/OTA tool rather than a bespoke updater. Buy the boring parts; build the scheduler, thermal coordinator, and yield router — those are the moat. The full control plane, scheduler, coordinator, and ledger design live in [`platform.md`](platform.md).

---

## 7. Demand layer — hybrid routing (Q4 marketplaces + Q5 own network)

This is the answer to "how do we get the GPUs rented," and it is explicitly **hybrid**. Full demand-routing and yield-router design lives in [`platform.md`](platform.md).

### 7.1 Marketplace connectors (instant fill — start here)

- **vast.ai host client first** — it has the deepest 4090 demand and a host model that fits distributed consumer hardware. This is the fastest path to revenue (Phase A, §11).
- Then **RunPod, Salad, io.net, Akash**. Salad is notable because it's explicitly designed for distributed/consumer hosts.
- Each connector **normalizes the marketplace's jobs into one internal job model** so the scheduler treats all demand sources uniformly.

### 7.2 Own demand (margin + control — build in parallel)

- **Direct/enterprise API** and **reserved-instance contracts** for customers who want capacity directly from Superheat (higher margin, avoids the 25–35% marketplace fee).
- **Interruptible batch / inference backlog** — Superheat-owned or partner workloads (e.g., batch inference, fine-tuning queues) that are *preemptible and have no deadline*. This backlog is the **utilization filler** (§8): it guarantees there is always something to run whenever a node has a thermal window.

### 7.3 The yield router

For each node-hour, the router picks the **highest expected $/GPU-hr** among the sources currently available to that node, subject to:

- the node's **thermal window** (can it run now / how long), from the thermal coordinator;
- **interruptibility** (don't put a non-preemptible premium job on a node about to lose its heat window);
- the node's **reliability tier** (§8.4);
- a **concentration guard** — honoring the risk register's **60% single-marketplace concentration trigger** from the market-research docs, so no single buyer can hold the fleet hostage on price or policy.

Net effect: marketplaces fill the easy demand instantly; our own backlog and contracts capture margin and prevent idle time; the router arbitrages between them every hour.

---

## 8. Driving utilization toward ~zero idle (Q5/Q6)

Idle GPU-hours are lost revenue *and* lost heat. Four mechanisms, layered (full utilization-engine design in [`platform.md`](platform.md)):

1. **Thermal battery + pre-heat (the key unlock).** Because the tank stores heat, compute does **not** have to wait for instantaneous heat demand — we can run the GPU now and bank the heat, or pre-heat ahead of a forecasted demand window. This decouples compute scheduling from real-time hot-water use and is what makes high utilization physically possible.
2. **Interruptible own-backlog filler (§7.2).** Whenever a node has a thermal window and no better-paying job, it runs the preemptible backlog. There is always *something* to run, so windows are never wasted.
3. **Reserved-instance floors.** A baseline of contracted/reserved demand provides revenue stability and a guaranteed minimum fill, distinct from volatile spot.
4. **Demand-response / VPP enrollment.** For the hours when *neither* compute nor heat is wanted, enroll the fleet's flexible load in demand-response / virtual-power-plant programs so the asset still earns. (This also appears in the market-research partner strategy — EnergyHub, Renew Home.)

### 8.4 Reliability tiering

Route work by what each node can promise:

- **High-reliability nodes** (strong uptime, good network, stable thermal demand — typically commercial) → premium, contracted, non-interruptible jobs.
- **Everything else** → spot/interruptible jobs and the batch filler, which tolerate preemption via checkpoint/resume (§9).

This maximizes total fleet revenue without over-promising on flaky nodes — and produces the clean, seasoned, high-reliability sub-pools that the financing engine wants.

---

## 9. Data I/O & renter UX (Q6)

On consumer links, **moving data is the bottleneck, not the GPU.** The data layer exists to hide that. Full data-I/O and renter-UX design lives in [`connectivity-and-data.md`](connectivity-and-data.md) (data) and [`platform.md`](platform.md) (renter API).

- **Regional content-addressed cache + pre-staging.** Cache popular base images, model weights, and datasets at regional points of presence (and on-node, §4.2). A renter's job pulls weights from a nearby cache, not over a home node's slow uplink. Content-addressing means we store each artifact once and dedup across the fleet.
- **Checkpoint / resume — mandatory.** Interruptible jobs *will* be preempted (heat window ends, node preempted for higher-yield work). Jobs must checkpoint to fast local NVMe and resume elsewhere. This is what makes interruptible filler (§8.2) and reliability tiering (§8.4) safe to offer.
- **Bandwidth-aware placement.** The scheduler measures each node's real upload/download and places data-heavy jobs on better-connected nodes; co-locates data and compute; routes bulk transfers over direct mesh paths rather than paid relays (§5).
- **Smooth renter entry.** A renter gets a node via the overlay with a one-command container/SSH workflow — they should not know or care that the GPU is in someone's basement. Persistent volumes with **clear eviction semantics** (what's kept, for how long, what to back up) so renters aren't surprised by preemption.

---

## 10. Security, multi-tenancy & trust

Full security/trust and legal/regulatory/compliance treatment lives in [`security-and-compliance.md`](security-and-compliance.md).

- **Three-way isolation.** Renter ⇄ homeowner ⇄ other renters are mutually isolated: tenant containers/microVMs can't reach the host, the thermal controller, the home LAN, or each other.
- **Egress controls + abuse enforcement.** Strict per-tenant egress firewall and ToS enforcement (no crypto mining, no illegal use, no abuse of the host's home network). Attestation that a node is running signed Superheat software before it's given paying work.
- **Honest limit: data confidentiality on consumer GPUs.** RTX 4090s lack the robust hardware TEE (confidential computing) that data-center GPUs increasingly offer. We **cannot** promise hardware-enforced confidentiality of a renter's data/model against a determined host on a standard node. Mitigations:
  - **Ephemeral & encrypted-at-rest** workloads, **no-persistence by default** (wipe on job end).
  - **Software attestation** of node image + agent.
  - A **"trusted-host" tier** (vetted commercial sites, possibly TEE-capable hardware) for sensitive workloads — priced higher and tracked separately. Be explicit with renters about which tier they're on.

---

## 11. Build phasing (recommended MVP → scale)

Sequenced for fastest path to revenue, then margin, then moat. Full roadmap lives in [`financial-and-roadmap.md`](financial-and-roadmap.md).

- **Phase A — Attach for revenue.** Node agent + immutable OS (A/B updates) + WireGuard overlay + telemetry + thermal guard + **vast.ai host attach**. Outcome: nodes earn on marketplaces with safe remote ops. This alone produces the metered tape Phase-1 financing needs.
- **Phase B — Own the orchestration.** Control-plane scheduler + thermal coordinator + **metering/billing ledger (securitization tape)** + interruptible batch backlog filler + reliability scoring/tiering. Outcome: margin capture and a real utilization floor.
- **Phase C — Own the demand & UX.** Own marketplace/API + reserved contracts + regional data caching layer + checkpoint/resume at scale + VPP/demand-response enrollment. Outcome: full hybrid yield routing and a smooth renter experience.

---

## 12. Open questions & risks

Full risk register lives in [`financial-and-roadmap.md`](financial-and-roadmap.md).

- **Confidential compute on 4090.** No strong TEE → limits the addressable market to non-sensitive workloads on standard nodes. Size the trusted-host tier; track TEE-capable hardware roadmaps.
- **CGNAT & upload bandwidth.** Relay cost and slow consumer uplinks cap data-heavy workloads and add opex. Quantify relay budget; consider per-region caches early; possibly cellular/secondary-link strategy for commercial sites.
- **Commercial SKU definition (8 vs 12 GPU).** Must be resolved before hardware freeze; it changes BOM, thermal design, and the underwriting tape (§3.3).
- **Refresh-reserve modularity.** The securitization model funds a **year-5 GPU module swap**. The node hardware/software must be designed for **modular GPU replacement** (and version-aware re-registration in the control plane) so the revenue curve can be re-upped for a second cycle.
- **Heat-demand seasonality.** Utilization assumptions depend on thermal overlap (70% commercial). Cold-climate vs warm-climate and summer DHW-only periods need per-region modeling so utilization promises to financiers are accurate.

---

## End-to-end walkthroughs

**Type:** Integrative / onboarding. **Authority:** NONE — this section introduces **no** new schemas, constants, or rules. It only narrates the existing design and points to the doc that owns each step. Where a fact has an owning doc, that doc wins; if a walkthrough ever disagrees with an owning doc, the owning doc is correct and the walkthrough is the bug.

**What this is for.** The system is split across the deep-dive docs above, each correct in isolation. This section stitches four *traces* through them so you can follow a single job, a single node-day, a single preemption, and a single node-lifetime end to end — and always know where to read more.

**The docs this stitches (read these for detail):**
this overview (the five principles, the five layers), [`data-contracts.md`](data-contracts.md) (NORMATIVE: job §4, telemetry §5, usage record §8, `fits()` §7, tiers §2/§3, thermal constants §6), [`platform.md`](platform.md) (connectors, yield router, settlement; registry, scheduler, thermal coordinator, reliability, ledger/tape; thermal window, four-layer stack, preemption; the direct renter surface), [`connectivity-and-data.md`](connectivity-and-data.md) (the outbound-only overlay; cache, checkpoint/resume, bandwidth-aware placement), [`node.md`](node.md) (the agent, OS, attestation; SKU, year-5 swap), [`security-and-compliance.md`](security-and-compliance.md) (isolation, no-TEE honesty), [`financial-and-roadmap.md`](financial-and-roadmap.md) (phasing, risk register), [`operations.md`](operations.md) (field ops, onboarding, node lifecycle in the field).

**A note on units & names** (all from `data-contracts.md` §1): durations in **seconds** (`_s`), power in **watts** (`_w`), energy in **kWh**, money in **USD**, GPU token is exactly `"RTX4090"`, tiers are `premium|standard|spot|probation`, and `T_GRACE_S = 30`.

### Orientation: the five principles and five layers every trace rides

Everything below is downstream of the five non-negotiable realities (§1): **(1) outbound-only** (nothing dials *into* the home router — see the overlay in Trace 1 step 8 and Trace 4 step 2); **(2) fail-safe to a water heater** (local safety always wins — Trace 2's `HALT_GPU`, Trace 4's HEATER-ONLY SAFE-MODE); **(3) compute is thermally coupled** (Trace 2 in full); **(4) measure then price reliability** (the score in Trace 2 step 5, the tier gate in Trace 1 step 5); **(5) hybrid demand** (the two job sources in Trace 1 step 1, unified at step 2).

A request descends, and a payout ascends, through the five layers (§2): **(5) demand** (connectors + yield router, `platform.md`), **(4) control plane** (scheduler, thermal coordinator, reliability, ledger, `platform.md`), **(3) connectivity** (the overlay, `connectivity-and-data.md`), **(2) node software** (the agent, `node.md`), **(1) node hardware** (the SKU + tank, `node.md`), with **data I/O** (`connectivity-and-data.md`) running alongside (5)→(1). Trace 1 is one full descent-and-ascent; the others zoom in.

---

### Trace 1 — A rental, end to end (the main one)

This is the canonical path: a job arrives (from a marketplace **or** the direct API), gets normalized, routed, placed, run, preempted, resumed, completed, metered, reconciled, and lands on the securitization tape. **Both the data flow and the money flow are made explicit** because they are deliberately separate pipelines (`platform.md` control-plane: telemetry is health, usage records are money).

#### 1.1 Numbered steps

1. **Demand arrives.** Either path produces the *same* internal object:
   - *Marketplace job:* a renter rents our listing on vast.ai/RunPod/Salad/etc.; the per-source **connector** pulls the assigned work and `normalize()`s it. — `platform.md` demand-routing Part A; connector interface §A.2.
   - *Direct/own-API job:* a direct renter `POST /v1/jobs` a `RenterJobSpec`; the control plane fills in `source:"direct"`, `tenant_class:"direct"`, `platform_fee_frac:0`. — `platform.md` renter-API §1, §2.4.
2. **Normalize to canonical `superheat.job.v1`.** Both inputs become the one nested job schema — `gpu_model:"RTX4090"`, `expected_duration_s` (seconds), `scheduling.interruptible`, `reliability_tier_required`, `min_thermal_window_s`, `economics.net_price_usd_per_gpu_hr`, `trust.confidentiality_required`. — `data-contracts.md` §4 (owns the schema); `platform.md` demand-routing §A.3; renter-API §1.2 (external→canonical mapping).
3. **Yield router picks the highest-net source for this node-hour.** It scores `E[$/gpu-hr]` across feasible sources, **honors reserved-floor commitments first**, drops infeasible candidates via the canonical `fits()` predicate, then applies the **concentration guard** (soft steering ≥ 50% share; **hard stop at the 60% ABS covenant**). The backlog filler is the always-available fallback so a window is never idle. — `platform.md` demand-routing Part C (router, §C.3 algorithm, §C.5 worked example); `fits()` owned by `data-contracts.md` §7; concentration cap is an ABS covenant per `financial-and-roadmap.md`.
4. **Router emits a `JobAssignment` to the scheduler.** The router decides *price*; the scheduler decides *placement feasibility and assignment*. The router never talks to nodes directly. — `platform.md` demand-routing §0; control-plane §5.
5. **Scheduler runs `fits()` against candidate nodes.** Hard filter: reliability tier (`tier_order(node) >= tier_order(job)`), confidentiality class, `gpu_count`, and the interruptible-vs-non-interruptible **thermal-window fit**. It pulls each node's open window from the thermal coordinator (`GetWindow`/`StreamWindows`). Only `active` + `attested` nodes are eligible. — `data-contracts.md` §7 (fits), §2 (tier rule); `platform.md` control-plane §5.2 (matching), §6.3 (window interface), §1.2 (lifecycle).
6. **Scheduler scores feasible nodes and reserves the GPU.** Score blends reliability, **data-locality**, thermal fit, net-quality, minus preemption risk and fragmentation. The winning `gpu` row flips `idle→renting` in a **single central transactional Postgres write** (never at the edge — that is what prevents double-booking). — `platform.md` control-plane §5.2, §9 (edge caches reads, reservations stay central).
7. **Data staged from the regional cache — not the slow uplink.** The scheduler issues a **prefetch hint**; the node agent diffs the job manifest against its on-node NVMe cache and pulls only the missing content-addressed chunks from the nearest **PoP** over the *download* direction (fast), assembling and hash-verifying them. The renter's slow uplink is never on this path. — `connectivity-and-data.md` data-I/O §2.4 (how a job pulls weights), §2.2 (cache hierarchy); placement co-locates data and compute (data-I/O §4.3).
8. **Container launched over the outbound-only overlay.** The scheduler `DISPATCH`es over the node's persistent outbound gRPC stream; the agent's `JobRuntime` launches the tenant container (NVIDIA Container Toolkit; egress-firewalled, no path to host/heater/home-LAN). A **session** peer is authorized for the renter (session subnet `100.65.x.x`), direct-path-preferred with DERP relay fallback. — `node.md` software §3.2, §4 (isolation); `connectivity-and-data.md` overlay §5 (connection establishment), §7 (segmentation).
9. **Job runs; telemetry flows.** The agent emits `superheat.telemetry.v1` every ~10 s (GPU util/temp/power, `thermal.mode`+`heat_demand_w`, `tank_temp_c`, `net.path`, `seq`, signed). Edge aggregators decimate 10 s→1 min for the cloud; this feeds dashboards and the **reliability score** — never billing. — `data-contracts.md` §5 (frame); `node.md` software §5.1 (producer); `platform.md` control-plane §2 (pipeline), §4 (scoring).
10. **Thermal window starts to close.** The thermal coordinator's window for this node shrinks (tank approaching `T_max`, or a forecasted draw needs the headroom). It pushes a window update; the scheduler sees the interruptible job's remaining window dropping toward `max(min_thermal_window_s, T_GRACE_S)`. — `platform.md` utilization-engine §2–§3 (window math), §6.1 (preempt triggers); control-plane §6.
11. **Graceful preempt (`T_GRACE_S = 30`).** Scheduler sends `PREEMPT`; the agent gives the interruptible job a **30 s** grace to flush its in-flight checkpoint to local NVMe, then SIGTERMs. (Local comfort safety can preempt instantly and unconditionally — the agent acts without the cloud.) — `data-contracts.md` §7 (`T_GRACE_S`, owned by `connectivity-and-data.md`); `platform.md` utilization-engine §6.2; `node.md` software §3.3 (local control loop wins).
12. **Checkpoint to local NVMe + async to PoP.** Checkpointing is three-stage and **decoupled from preemption**: Stage 1 writes NVMe (fast, always succeeds); Stage 2 uploads *delta chunks* to the PoP asynchronously *during* compute; Stage 3 registers durability in the control plane. Because 30 s cannot finish a multi-minute upload, the **durable resume source is the last PoP-persisted checkpoint**, not the in-flight flush. — `connectivity-and-data.md` data-I/O §3 (RTO ≤ 5 min, RPO ≤ 15 min); `data-contracts.md` §7 (durable path note).
13. **Resume on another eligible node.** Scheduler `REQUEUE`s; placement picks a new node with a window; that node **downloads** the checkpoint + weights from the PoP (fast direction) and resumes at the durable step. To the renter it is a brief stall, not a failure — the brokered session tunnel re-points. — `platform.md` control-plane §5.3 (job lifecycle); `connectivity-and-data.md` data-I/O §3.3, overlay §7.3; renter view in `platform.md` renter-API §4.2.
14. **Completion.** Job exits 0 → `COMPLETED`. Tenant scratch is wiped (per-job key dropped, TRIM) — no-persistence by default. — `platform.md` control-plane §5.3; `security-and-compliance.md` (wipe-on-end); `connectivity-and-data.md` data-I/O §6.
15. **Signed usage record emitted (`node_sig`).** The agent emits one `superheat.usage.v1` per (GPU slot, job, billing period) with the metered facts (`gpu_slot`, `gpu_seconds`, `energy_kwh`, `gross/fee/net_usd`) and signs it with the node identity key. **Per-GPU, never spanning a billing-period boundary.** — `data-contracts.md` §8 (primitive); `node.md` software §5.2 (producer); `platform.md` control-plane §7.1.
16. **Reconciled vs marketplace settlement.** Per source, the connector `pull_settlement()` returns realized payouts; summed signed-usage `net_usd` must reconcile to the marketplace statement within the ± tolerance. (Direct jobs skip the marketplace leg.) The ± tolerance is a **money** check, never a telemetry check. — `platform.md` demand-routing Part D; control-plane §7.4; `data-contracts.md` §8.
17. **Cloud counter-signature (`cloud_sig`).** On reconcile, the control plane counter-signs the usage record, completing the double-signed provenance chain. — `data-contracts.md` §8; `platform.md` control-plane §7.1.
18. **Owner settlement + platform take.** Reconciled net is split per the node's financing/tranche tag: owner payout, Superheat servicing/platform fee (25% marketplace-equivalent take), electricity-offset note. — `platform.md` control-plane §7.2 (settlement table), §7.4; demand-routing §D.2.
19. **Row lands on the securitization tape.** The `securitization_tape` materialized view aggregates per-GPU usage (joined on `node_id` AND `gpu_slot`), rolled to one node-row-per-period: GPU-hours, realized $/GPU-hr, revenue-by-source, reliability/seasoning. Only `active` + `seasoned` nodes appear. **The metered ledger *is* the tape.** — `platform.md` control-plane §7.3.

#### 1.2 Sequence diagram — data flow and money flow

```
Renter/        Connector/      Yield      Scheduler   Thermal    Node Agent     PoP        Ledger/
Marketplace    Direct API      Router      (cp §5)    Coord §6   (node)         Cache      Tape (cp §7)
   │               │             │           │          │           │            │            │
   │ rent / POST   │             │           │          │           │            │            │
   │──────────────►│ normalize   │           │          │           │            │            │
   │               │ →job.v1     │           │          │           │            │            │
   │               │ (dc §4)─────►│ pick max  │          │           │            │            │
   │               │             │ net $;    │          │           │            │            │
   │               │             │ fits()+   │          │           │            │            │
   │               │             │ conc.guard│          │           │            │            │
   │               │             │──assign──►│ place()   │           │            │            │
   │               │             │           │─GetWindow►│           │            │            │
   │               │             │           │◄─window───│           │            │            │
   │               │             │           │ fits()+score; reserve GPU (central txn)         │
   │               │             │           │─prefetch hint──────►│ pull chunks (DOWN, fast)  │
   │               │             │           │           │         │◄──────────────│ weights   │
   │               │             │           │─DISPATCH (outbound overlay)──►│ launch container │
   │◄═ session over overlay (direct-pref, relay fallback; connectivity-and-data overlay §5/§7) ═►│ │
   │               │             │           │           │   RUNNING; telemetry 10s ─► cp §2/§4 │
   │               │             │           │           │ window closing            │            │
   │               │             │           │◄─window update (preempt)──│            │            │
   │               │             │           │─PREEMPT──────────────►│ flush ckpt    │            │
   │               │             │           │           │           │ (grace 30s)──►│ delta up   │
   │               │             │           │           │           │── usage rec ──────────────►│ node_sig
   │               │             │           │ REQUEUE → resume elsewhere            │            │
   │               │             │           │─DISPATCH (node B)─────────────►│ DOWN ◄│ ckpt+wts   │
   │◄═ session re-points to node B (brief stall, renter-API §4.2) ══════════►│ RESUME @ step n    │
   │               │             │           │           │       COMPLETED; wipe scratch          │
   │               │             │           │           │           │── usage rec ──────────────►│
   │               │ ◄ pull_settlement (D) ──────────────────────────────────────────────────────│
   │               │             │      reconcile usage net_usd vs settlement (±tol; cp §7.4)     │
   │               │             │           │           │           │      cloud_sig (dc §8)─────►│
   │               │             │           │   owner settlement + platform take (cp §7.2) ──────►│
   │               │             │           │           │           │        tape row (cp §7.3)──►│
```

**Data flow (down/up):** weights and checkpoints ride the cache hierarchy — pulled *down* from the PoP (fast), pushed *up* to the PoP as deltas, asynchronously. The home uplink is minimized. Bulk traffic is kept *off* paid relays (caches serve the fast down-link; checkpoint dedup ships only changed chunks), which is also the lever that holds the relay-opex budget down. (`connectivity-and-data.md` data-I/O §1–§4; relay-cost partnership overlay §6.)
**Money flow (signed/reconciled):** `node_sig` at delivery → reconcile vs marketplace settlement → `cloud_sig` → owner/platform split → tape row. Telemetry never touches this path; the ± reconciliation tolerance is a money check against marketplace settlement, not a tolerance on lossy telemetry. (`platform.md` control-plane §0, §7.4; `data-contracts.md` §8.)

> **A worked routing moment** (so the numbers in step 3 are concrete): on a `standard` H1 with a 45-min window, the router scores vast.ai at ~$0.226/GPU-hr expected net vs RunPod ~$0.167 and direct ~$0.155 (high net price but thin `p_fill` early), so vast.ai wins — *unless* vast.ai is already ≥ 60% of trailing-30d revenue, in which case the hard concentration stop drops it and RunPod wins at $0.167 rather than idling. A pending reserved commitment (net ~$0.49) would have been placed at step 0 before the auction ran. — `platform.md` demand-routing §C.5.

---

### Trace 2 — A thermally-driven day in one node

This trace follows **one H1 home node** (single RTX 4090, ~412 W, 80-gal tank, `E_store ≈ 8.8 kWh`) across a winter day to show that scheduling is a *joint compute-and-heat* decision. The thermal coordinator owns the math (`platform.md` utilization-engine §2–§3); the on-node control loop enforces safety locally (`node.md` software §3.3). The single most important fact for the whole business lives here: because the tank is a **thermal battery**, "heat wanted now" is not the binding constraint — "wanted now **or** storable" is, and that wider window is what makes high utilization (the economics memo's 60%×70% *commercial* base case) physically possible. — `platform.md` utilization-engine §1, §3.3.

#### 2.1 Numbered steps

1. **Morning hot-water draw (realtime heat).** 06:00–08:00 DHW + space-heat peak. `thermal.mode = "realtime"`; `D(t) > P_in`. The GPU runs flat-out and *every watt of its heat is consumed instantly* — this is the free-electricity (discharge-led) regime. The yield router fills the window with the best-paying job (L2/L3). — `platform.md` utilization-engine §2.2 (discharge-led), §9 (worked day).
2. **Midday pre-heat / banking compute into the tank (storing).** Demand drops; `thermal.mode = "storing"`. Rather than idle, the coordinator runs the GPU to **bank heat** into `E_store`, and **pre-heats ahead of the forecasted evening peak** — keeping headroom in reserve so the peak is served by compute heat, not the resistive element. `storage_headroom_kwh` falls as the tank charges. — `platform.md` utilization-engine §2.3 (pre-heat), §2.2 (charge-led); coordinator interface control-plane §6.2.
3. **Tank full → no compute (or VPP/DR).** If the band saturates (`headroom → 0`) with no draw and no profitable job, the local control loop `HALT_GPU`s ("nowhere for heat to go"). This is the `IDLE_AND_COAST` state the engine minimizes — so the node instead earns on **L4 demand-response / VPP** if an event is active (a DR *charge* event is a free pre-heat; a DR *shed* is GPU-off, the comfort-safe default). — `platform.md` utilization-engine §4 (L4), §8 (control loop), §9 (summer contrast); `node.md` software §3.3 (`tank_has_storage_headroom`).
4. **Evening draw.** 18:00–22:00 peak; back to discharge-led/realtime. The heat banked at midday plus live compute heat cover the draw; run flat-out with the best job. — `platform.md` utilization-engine §9.
5. **Reliability scoring over the day.** Every telemetry frame feeds the trailing-window reliability score (`uptime`, `thermal_avail`, `net_quality`, `preempt_penalty`, `fault_penalty`), recomputed at least daily and EWMA-smoothed into a tier. A day of clean uptime + high thermal availability nudges the score up; the node must also be **seasoned (≥ 30 d)** to be tape-eligible. — `data-contracts.md` §3 (formula), §2 (tiers); `platform.md` control-plane §4.

> **Throughout:** local safety always wins. `T_tank > T_max`, lost thermal link, lost coolant flow, or lost heartbeat each force `HALT_GPU` on the node's 1 Hz control loop, regardless of what the cloud asked. The customer always has hot water (Principle 2). — `node.md` software §3.3, §3.4 (HEATER-ONLY SAFE-MODE); §4.2 of this overview.

#### 2.2 Sequence diagram — a node-day

```
 Time   Thermal mode   Tank SoC      Engine action (util-engine §8)        Layer    Heat status
 ─────  ────────────   ──────────    ───────────────────────────────────   ─────    ───────────
 00–05  storing/none   charging↑     run if job; pre-heat for 06:00 peak    L2/L1    banked (charge)
 06–08  realtime       discharging↓  run flat-out; heat consumed instantly  L2/L3    FREE (discharge)
 08–17  storing/realtime tracking    run continuously; charge/discharge mix L2 spot  mostly free
                                      filler (L1) fills any spot gaps
 17     preheat        hold headroom  reserve band for evening peak          (hold)   —
 18–22  realtime       discharging↓  run flat-out; banked + live heat        L2/L3    FREE (discharge)
 22–24  storing        charging↑     run if job; else DR-charge              L1/L4    banked / DR

   each interval: agent emits superheat.telemetry.v1 (dc §5) ──► edge aggregator (cp §2)
                  ──► reliability score recompute (dc §3, cp §4) ──► tier ──► tape track record (cp §7.3)

   ANY interval, hard trip (T_max / no flow / link lost):
       node control loop (node §3.3) ─► HALT_GPU ─► HEATER-ONLY if needed  (cloud not consulted)
```

> Note the regime flip across the year: in winter `E_day > GPU_max_daily`, so the thermal constraint is **loose** and utilization is demand-limited (target ≥ 60%, defended by the L1 filler). In summer/warm-climate DHW-only, the GPU **overproduces**, the tank saturates, and the node duty-cycles down / leans on DR — the honest seasonal floor the 60%×70% *commercial* average is built to absorb. — `platform.md` utilization-engine §3.3, §7, §9.

---

### Trace 3 — A preemption + resume close-up

Zooms into steps 10–13 of Trace 1 from three viewpoints simultaneously: **what the renter sees**, **what the agent does**, **what the scheduler does** — plus the RPO/RTO contract.

**What the renter literally experiences:** they ran an `--interruptible` (spot-tier) job via `superheat run` and stayed attached over `superheat ssh`. At preemption their terminal stalls briefly; `superheat logs -f` shows the gap; an SDK `for ev in job.events()` loop yields `job.preempted` → `job.resuming` → `job.running`; `GET /v1/jobs/{id}` shows `preemptions` incrementing and `last_checkpoint.durable: true`. The endpoint URL never changes. This is the contract spot renters opted into when they chose the cheapest tier. — `platform.md` renter-API §3.1 (CLI), §3.2 (SDK), §4 (interruption UX); `connectivity-and-data.md` data-I/O §5.1 (stable session).

#### 3.1 Numbered steps

1. **Trigger (scheduler/coordinator).** One of: thermal-window expiry (planned), yield preemption (a higher-value L2/L3 job wants the node), DR shed, or local comfort-safety (instant, local). Priority order and "comfort always wins" are fixed. — `platform.md` utilization-engine §6.1.
2. **Renter is notified.** A `job.preempted` event (and webhook) fires with `reason ∈ {thermal_window_closing, higher_yield, node_fault, operator}` and `expected_resume_by_s` (RTO target). For a checkpointed job this is **not** a failure. — `platform.md` renter-API §2.7, §4.2.
3. **Agent flushes the in-flight checkpoint.** `JobRuntime` gets `PREEMPT`, gives the interruptible job **`T_GRACE_S = 30`** to flush to local NVMe, then SIGTERMs the container and closes the heat path (GPU OFF/THROTTLE). — `node.md` software §3.3, §5.2; `platform.md` utilization-engine §6.2.
4. **Durability is what matters, not the flush.** The job is "safely resumable up to step n" only once Stage 3 (PoP-durable registration) for step n completed — which happened *asynchronously during compute*. The 30 s flush is best-effort; the guaranteed resume source is the last PoP-durable checkpoint. — `connectivity-and-data.md` data-I/O §3.2; `data-contracts.md` §7.
5. **Scheduler requeues and re-places.** `RUNNING → PREEMPTED → (checkpoint) → QUEUED`, then placement on a new eligible node (bandwidth-aware: big working sets go to A-tier-uplink nodes). — `platform.md` control-plane §5.3; `connectivity-and-data.md` data-I/O §4.3.
6. **Node B resumes.** Node B *downloads* the durable checkpoint + weights from the PoP (fast direction) and resumes at step n. The brokered session tunnel re-points so the renter's endpoint URL is stable. — `connectivity-and-data.md` data-I/O §3.3, overlay §7.3.
7. **RPO/RTO contract.** **RTO ≤ 5 min** (p50 ≤ 2 min, p95 ≤ 10 min) for working sets ≤ 40 GB; **RPO ≤ one checkpoint interval** (default ≤ 15 min of compute). Surfaced to the renter as `resume.rto_target_s` / `rpo_bound_s`. — `connectivity-and-data.md` data-I/O §3.1; `platform.md` renter-API §2.5, §4.4.

> **Opaque jobs:** a job that does not use framework/SDK checkpointing gets best-effort CRIU/`cuda-checkpoint` and may restart from scratch — so it is steered to non-interruptible higher-reliability nodes or priced accordingly. — `connectivity-and-data.md` data-I/O §3.5; `platform.md` renter-API §4.2.

> **Why "preemption is a migration, not a loss":** the expensive direction (node→PoP upload) already happened asynchronously during compute; resume only needs the cheap direction (PoP→node B download). That asymmetry is the whole reason a 5-min RTO is achievable on slow consumer uplinks, and it is why the L1 interruptible filler and reliability tiering are *safe to sell* in the first place. — `connectivity-and-data.md` data-I/O §3.1; `platform.md` utilization-engine §6.2.

#### 3.2 Sequence diagram — three viewpoints on one preemption

```
 RENTER (renter-API)         SCHEDULER (cp §5)        AGENT / Node A (node)        PoP        Node B
   │                              │                          │                       │           │
   │           ... job RUNNING; checkpoints uploading async during compute ...       │           │
   │                              │                          │ ckpt step n → NVMe    │           │
   │                              │                          │ async delta upload ──►│ (durable) │
   │                              │ ◄── register durable(n) ──┤                       │           │
   │                              │ thermal window closing OR higher-yield job arrives│           │
   │ ◄ job.preempted (reason,     │── PREEMPT ──────────────►│                       │           │
   │   expected_resume_by_s) ─────│                          │ grace ≤ 30s: flush ──►│ (best-eff)│
   │   webhook + event stream     │                          │ SIGTERM; GPU OFF      │           │
   │                              │ ◄── stopped @ durable n ──┤                       │           │
   │                              │ REQUEUE → place (bw-aware)│                       │           │
   │ ◄ job.resuming ──────────────│── DISPATCH ─────────────────────────────────────────────────►│
   │                              │                          │                       │ ckpt n ──►│ DOWN
   │                              │                          │                       │ weights ─►│ DOWN
   │ ◄ session tunnel re-points (connectivity-and-data overlay §7.3); brief stall, stable URL ───►│
   │ ◄ job.running ───────────────│                          │                       │  RESUME @ step n
   │                              │                          │                       │           │
   │   RTO ≤ 5 min (p50 ≤ 2) · RPO ≤ 1 ckpt interval (≤ 15 min)   — connectivity-and-data §3.1, renter-API §4.4
```

---

### Trace 4 — A node's life

Follows one physical unit from carton to second revenue cycle. Lifecycle states are owned by the registry (`platform.md` control-plane §1.2); install/swap mechanics by `node.md` hardware §8 and field ops by `operations.md`; on-node provisioning/attestation by `node.md` software §8.

#### 4.1 Numbered steps

1. **Install.** A plumber + electrician install the heater-embedded node; H1 shares the existing **240 V / 30 A** water-heater branch, onboard 2.5 GbE to the home router, **no inbound ports / no router config** (a hard installability requirement). The node is `provisioned` in the registry. — `node.md` hardware §6–§7 (electrical/network interfaces); `operations.md` (field install/onboarding); `platform.md` control-plane §1.2 (`provisioned → installed`).
2. **Attest.** First boot enters `PROVISIONING`: the agent uses a one-time enrollment token, the hardware-bound key (TPM/secure element) registers its public key, dials *out* to the headscale coordinator, and proves it runs genuine signed Superheat software (image digest ∈ allowed set). On pass the node becomes `attested` and may receive **paying** work; only attested image+identity node-hours enter the tape. — `node.md` software §8 (attestation flow), §3.4 (state machine); `connectivity-and-data.md` overlay §5/§8 (overlay join + identity); `platform.md` control-plane §1.2 (`installed → attested`).
3. **Active / earning.** The node goes `active`, heartbeats telemetry, accrues a reliability score, and — once **seasoned (≥ 30 d)** — becomes financeable collateral on the tape. It now runs Traces 1–3 every day: routed jobs, thermally-gated compute, preempt/resume, signed usage records. — `platform.md` control-plane §1.2 (`active`), §4 (scoring), §7 (tape); `data-contracts.md` §2/§3 (seasoning gate).
4. **Year-5 GPU swap.** The financing model funds a refresh reserve. A certified installer does a tool-light, quick-release **GPU module swap** (dry-disconnect coolant coupler — no re-plumbing) — e.g., 4090 → 5090-class with no electrical change. The unit goes `maintenance` for the swap. — `node.md` hardware §8.1 (modular swap); `operations.md` (field swap procedure); `platform.md` control-plane §1.2 (`active → maintenance`); `financial-and-roadmap.md` R4.
5. **Re-registration (version-aware).** The agent detects the changed card, **re-attests**, and the registry does a version-aware re-registration: the old `gpu` row is retired (`retired_at` set, serial kept for tape lineage) and a fresh card row takes the slot — without blocking the slot reuse. The collateral generation change shows cleanly on the tape (a securitization covenant). — `node.md` hardware §8.2; `platform.md` control-plane §1.1 (`gpu` table, one-live-per-slot index), `RegisterGpuSwap`; `connectivity-and-data.md` overlay §8 (a swap re-provisions a fresh identity).
6. **Continued earning (second cycle).** Re-attested and back to `active`, the node re-ups its revenue curve for a second ~5–6-year cycle on newer silicon — the same Traces 1–3 resume. — `node.md` hardware §8; `financial-and-roadmap.md` R4.

> **Off-ramps at any time:** `active ⇄ degraded` (reliability score crosses a tier boundary), `quarantined` (OTA halt or abuse detection), and ultimately `retired`. A failed attestation, anomaly, or decommission triggers **instant overlay revocation** (key removed from coordinator policy). And under any software failure the unit drops to **HEATER-ONLY SAFE-MODE** — still a working water heater. — `platform.md` control-plane §1.2; `connectivity-and-data.md` overlay §8.3; `node.md` software §3.4, §6 (recovery ladder).

#### 4.2 Sequence diagram — node lifecycle

```
  carton
    │  plumber + electrician; 240V/30A branch; 2.5GbE; NO inbound (node-hw §6-7; operations field install)
    ▼
 [provisioned] ──► [installed] ──► first boot: PROVISIONING (node sw §3.4)
                                      │  TPM key + enrollment token; dial OUT to headscale (conn §5/§8)
                                      │  attest: image_digest ∈ allowed_set (node sw §8)
                                      ▼
                                 [attested] ──── eligible for PAID work (cp §1.2)
                                      │  seasoning ≥ 30d ⇒ tape-eligible (dc §2/§3)
                                      ▼
            ┌──────────────────► [active] ⇄ [degraded]   (runs Traces 1–3 daily)
            │                        │   │        └─ score crosses tier boundary (cp §4)
            │  re-attest + back      │   └─ [quarantined] ← OTA halt / abuse (cp §3, security-and-compliance)
            │  to earning            │
            │                        ▼  year-5 refresh-reserve swap
            │                   [maintenance] ── quick-release GPU module swap (node-hw §8.1; operations)
            │                        │   agent detects new card → re-attest (node sw)
            │                        │   registry: retire old gpu row (lineage), fresh card in slot
            │                        │   version-aware re-registration (node-hw §8.2; cp §1.1)
            └────────────────────────┘
                                      │  decommission / unrecoverable
                                      ▼
                                  [retired] ── overlay key revoked (conn §8.3); old serial kept on tape

  ANY state, software failure ─► HEATER-ONLY SAFE-MODE (node sw §3.4) ─► customer keeps hot water
```

---

### Following one dollar (money flow, slowly)

For the finance-minded reader, the same path as Trace 1 steps 15–19, framed as the journey of a single GPU-hour's revenue — this is the chain that makes the ledger securitizable:

1. The buyer pays gross $/GPU-hr to the marketplace (or directly to Superheat for `direct`). — `data-contracts.md` §4 `economics`.
2. At delivery the node emits a **per-GPU**, period-bounded `superheat.usage.v1` with `gross/fee/net_usd` and signs it (`node_sig`). This is the *only* billing primitive; telemetry is never billed. — `data-contracts.md` §8; `node.md` software §5.2.
3. The connector pulls the lagged marketplace settlement; summed signed-usage `net_usd` must reconcile to the statement within ± tolerance (orphans/missing/variance are flagged, not silently absorbed). — `platform.md` demand-routing §D.2.
4. On reconcile the control plane writes `cloud_sig` — now the GPU-hour is non-repudiable by *both* node and platform. — `data-contracts.md` §8; `platform.md` control-plane §7.1.
5. Net is split per the node's `tranche_id`: owner payout (gross − 25% platform fee), servicing fee, electricity-offset note. — `platform.md` control-plane §7.2, §7.4.
6. The `securitization_tape` view aggregates per (node, gpu_slot, month) — joined on **both** `node_id` AND `gpu_slot` so an 8-GPU node is not Cartesian-multiplied — and emits one node-row-per-period with GPU-hours, realized $/GPU-hr, source mix (with the concentration share the guard enforced), reliability/seasoning. Only `active` + `seasoned` rows appear. That row is the credit file an ABS underwriter rates. — `platform.md` control-plane §7.3; concentration covenant `financial-and-roadmap.md`.

### How the traces compose

- **Trace 1** is the spine; **Trace 2** is what gates *when* Trace 1 can run on a given node (the thermal window); **Trace 3** is the zoom-in on Trace 1's preempt/resume; **Trace 4** is the envelope all of them run inside (a node must be `attested`+`active` to appear in Trace 1's candidate set, and `seasoned` to appear on the tape).
- **Two pipelines, always separate:** the *data* path (cache pulls down, checkpoint deltas up — `connectivity-and-data.md`) and the *money* path (signed usage records → reconcile → counter-sign → tape — `platform.md` control-plane §7, `data-contracts.md` §8). Telemetry (`data-contracts.md` §5) feeds neither billing nor placement reservations; it feeds health/scoring only.
- **One predicate everywhere:** `fits()` (`data-contracts.md` §7) is the single feasibility check used by the yield router (`platform.md` demand-routing §C.3), the scheduler (`platform.md` control-plane §5.2), and the utilization engine (`platform.md` utilization-engine §5.2).

---

### Inconsistencies spotted

These are observations surfaced while stitching; this section does **not** resolve them (per its non-authoritative status). Items previously marked RESOLVED have been dropped.

1. **`T_GRACE_S` vs the renter-visible RPO look inconsistent but are not.** `T_GRACE_S = 30 s` (`data-contracts.md` §7) is the *flush* budget, while RPO is "≤ one checkpoint interval, default ≤ 15 min" (`connectivity-and-data.md` data-I/O §3.1). A new reader can mistake 30 s for the lost-work bound. The docs are internally consistent (the durable source is the last PoP checkpoint, not the 30 s flush — both say so), but the relationship is only spelled out in prose; it would help to state it as a one-line invariant in `data-contracts.md` §7.

2. **Commercial SKU power: `heater_kw` comment vs the disputed pitch figure.** `platform.md` control-plane §1.1 annotates the commercial SKU as "~3.3 GPU-kW … NOT the disputed 33 kW pitch figure," and `node.md` flags 8× vs 12× GPU as unresolved before hardware freeze. Not a contradiction *within* the system docs (they consistently use the 8×/3.3 kW underwritten number), but the 8-vs-12 SKU decision is still open and every trace here assumes the 8× commercial / 1× H1 SKUs. Flagged so Trace 1/2 numbers are read against the underwritten SKU, not the pitch SKU. (See also §3.3 of this overview.)

3. **`reserved` exemption from the concentration guard while requiring `standard+`.** `platform.md` demand-routing §C.3 exempts `direct|reserved|internal_backlog` from the concentration guard, and §B.1 says reserved is non-interruptible requiring `reliability_tier_required >= standard`. Consistent, but note the interaction: reserved demand both bypasses the cap *and* consumes premium/standard node-hours first (§C.4) — so a reserved-heavy month tightens premium-node availability for spot. Not an inconsistency, just a coupling reviewers should be aware of; no doc states it explicitly.

---

*Sources this overview is consistent with:* `../market-research/Superheat AI Data Center.md`, `../market-research/GPU_Pricing_Research_4090_H100.md`, `../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md`, `../market-research/Superheat_Compute-Network_Debt-Engine_Onepager.md`, `../market-research/Superheat_Partnership_and_Capital_Map.md`.
