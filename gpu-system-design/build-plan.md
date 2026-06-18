# Superheat — Build Plan (Actionable Playbook)

This document is the **do-it version** of the design. The eight design docs in this same folder (`node.md`, `platform.md`, `connectivity-and-data.md`, `data-contracts.md`, `security-and-compliance.md`, `financial-and-roadmap.md`, `operations.md`, `system-design-overview.md`) are the *what and why* (architecture, schemas, rationale); this playbook is the *what to do, in order* — concrete, checkable steps to set up and build the whole system.

> If a step needs detail, follow the `design-doc §` pointer on that step. Don't re-read the whole design to act — the playbook tells you the next move and where to look only if you're stuck.

## The path at a glance

```
  DECISIONS + SETUP            PHASE A                 PHASE B                  PHASE C
  (Part 1)                     (Part 2)                (Part 3)                 (Part 4)
  lock 8 decisions       →     attach for revenue  →   own orchestration   →    own demand & UX
  stand up env/repos           node+OS+overlay         scheduler+thermal        own API+SDK
  procure pilot hw             telemetry+registry      authoritative tape       regional cache
                               vast.ai connector       yield router v1          reserved+VPP/DR
                               metering v0 (read-only)  checkpoint/resume v1     trusted-host tier
  ───────────────────────────────────────────────────────────────────────────────────────────
  GATE: SKU frozen,      EXIT: ≥1,000 units,     EXIT: owned ledger        EXIT: ≥10k seasoned
  fail-safe proven,            12-mo clean tape,       reconciles, DSCR          units, ABS-ready,
  pilot site + hw              A/B rollback proven,    ≥2.5×, ≥30% contracted,   contracted share
                               concentration <60%      warehouse pre-read        rising, backup
                                                                                 servicer tested
  ───────────────────────────────────────────────────────────────────────────────────────────
  FINANCING:  equity pilot →   venture debt/        →  warehouse facility   →    inaugural ABS →
                               equipment lines                                   programmatic
```

Each phase's **technical exit criteria are the next financing rung's entry criteria** (see [`financial-and-roadmap.md`](financial-and-roadmap.md) Part C). Build in the order the capital needs it.

## Two rules that override everything

1. **Fail-safe to a water heater is a hard gate, not a feature.** No fleet rollout in any phase until the HIL safety suite + A/B-rollback proof + zero-hot-water release gate pass (Phase A steps 25–28, see [Part 2 — Workstream 6](#workstream-6--safety--release-gating-hard-gates--block-fleet-rollout)). A compute fault may never cost a customer hot water.
2. **The metered ledger is the product.** Every phase is "done" only when it produces the financial artifact the next funding rung underwrites — clean per-unit tape, an authoritative reconciling ledger, a ratable seasoned pool. Code shipping ≠ phase done.

## Decision gates before you write Phase A code

Five of the eight decisions in [Part 1 — Decisions & setup](#part-1--decisions--setup) are hard Phase-A blockers — most importantly **freeze the 8× RTX 4090 commercial SKU** (it sets the BOM, the thermal envelope, and the securitization tape) and **prove the fail-safe recovery path**. Don't start fleet hardware or origination until these clear.

---

## Table of Contents

- [Part 1 — Decisions & setup](#part-1--decisions--setup)
  - [1.1 — Blocking decisions to make first](#11--blocking-decisions-to-make-first)
  - [1.2 — Foundational setup to start engineering](#12--foundational-setup-to-start-engineering)
    - [A. Team / roles](#a-team--roles-to-assign-or-hire-phase-a-core--57-eng)
    - [B. Repos / monorepo](#b-repos--monorepo-to-create)
    - [C. Tech-stack picks](#c-tech-stack-picks-the-design-already-implies-these--adopt-dont-re-debate)
    - [D. External accounts / vendors](#d-external-accounts--vendors-to-set-up)
    - [E. Pilot — site + hardware + bench](#e-pilot--1-commercial-site--hardware--bench)
    - [Phase-A "ready to build" gate](#phase-a-ready-to-build-gate-when-part-1--part-2-are-done)
- [Part 2 — Phase A: Attach for revenue](#part-2--phase-a-attach-for-revenue)
  - [Workstream entry gate](#workstream-entry-gate-resolve-before-any-phase-a-build)
  - [Workstream 1 — Node OS + Agent](#workstream-1--node-os--agent)
  - [Workstream 2 — Connectivity](#workstream-2--connectivity)
  - [Workstream 3 — Control-plane skeleton](#workstream-3--control-plane-skeleton)
  - [Workstream 4 — Marketplace Attach](#workstream-4--marketplace-attach)
  - [Workstream 5 — Metering v0 (read-only tape)](#workstream-5--metering-v0-read-only-tape)
  - [Workstream 6 — Safety / Release-gating (HARD GATES)](#workstream-6--safety--release-gating-hard-gates--block-fleet-rollout)
  - [Phase A Exit Checklist](#phase-a-exit-checklist--phase-1-financing-entry)
- [Part 3 — Phase B: Own the orchestration](#part-3--phase-b-own-the-orchestration)
  - [Workstream 0 — Foundations](#workstream-0--foundations-do-first-everything-else-depends-on-these)
  - [Workstream 1 — Scheduler / Orchestrator](#workstream-1--scheduler--orchestrator-the-operational-core)
  - [Workstream 2 — Thermal Coordinator](#workstream-2--thermal-coordinator-tank-as-battery-pre-heat)
  - [Workstream 3 — Authoritative Ledger / Securitization Tape](#workstream-3--authoritative-ledger--securitization-tape--money)
  - [Workstream 4 — Reliability Scoring + Tiering](#workstream-4--reliability-scoring--tiering)
  - [Workstream 5 — Yield Router v1](#workstream-5--yield-router-v1-with-the-60-concentration-guard)
  - [Workstream 6 — Backlog Filler](#workstream-6--interruptible-batchinference-backlog-filler)
  - [Workstream 7 — Checkpoint / Resume v1](#workstream-7--checkpoint--resume-v1)
  - [Workstream 8 — DR / VPP Enrollment](#workstream-8--dr--vpp-enrollment-begin)
  - [Workstream 9 — Reconciliation + Release Gates](#workstream-9--reconciliation--release-gates-phase-b-financial-gate--money)
  - [Phase B Exit Checklist](#phase-b-exit-checklist--criterion--verifying-step)
- [Part 4 — Phase C: Own the demand & UX](#part-4--phase-c-own-the-demand--ux)
  - [Workstream 1 — Renter API + SDK](#workstream-1--renter-api--sdk-the-direct-demand-surface)
  - [Workstream 2 — Direct / reserved demand](#workstream-2--direct--reserved-demand-own-demand--concentration-dilution)
  - [Workstream 3 — Regional cache + data layer](#workstream-3--regional-content-addressed-cache--data-layer-the-relay-cost-lever)
  - [Workstream 4 — Checkpoint/resume at scale](#workstream-4--checkpointresume-at-scale--bandwidth-aware-placement)
  - [Workstream 5 — VPP / Demand-Response at fleet scale](#workstream-5--vpp--demand-response-at-fleet-scale-the-seasonal-floor)
  - [Workstream 6 — Trusted-host confidentiality tier](#workstream-6--trusted-host-confidentiality-tier-sizes-the-r1-tam-ceiling)
  - [Workstream 7 — Renter UX](#workstream-7--renter-ux-data-center-ergonomics-on-a-basement-gpu)
  - [Workstream 8 — Backup-servicer drill](#workstream-8--backup-servicer-drill-the-deal-clearing-r7-cure)
  - [Workstream 9 — Collateral scale, seasoning & covenant reporting](#workstream-9--collateral-scale-seasoning--covenant-reporting-the-abs-readiness-backbone)
  - [Workstream 10 — Cross-cutting hardening](#workstream-10--cross-cutting-hardening-modular-refresh-residency-fleet-scale-validation)
  - [Phase C Exit Checklist](#phase-c-exit-checklist--criterion--verifying-step)
- [Part 5 — Master checklist](#part-5--master-checklist)

---

## Part 1 — Decisions & setup

**Status:** Actionable checklist · **Date:** June 2026 · **Audience:** Founders, eng leads, finance/legal, program mgmt
**Purpose:** The things to **lock down and stand up BEFORE / at the start of building**. Two parts:
- **Section 1.1 — Blocking decisions** that must resolve before they unblock work downstream.
- **Section 1.2 — Foundational setup** to get the working environment, repos, vendors, and pilot stood up.

**Five principles (the invariants every choice below must respect):** (1) outbound-only networking · (2) fail-safe-to-heater · (3) thermally-coupled compute · (4) measure-then-price reliability · (5) hybrid demand. *(`system-design-overview.md` §1)*

**Build order:** Phase A (attach to marketplaces → revenue + tape) → Phase B (own orchestration + ledger) → Phase C (own demand + data + VPP). *(`financial-and-roadmap.md` Part C; `system-design-overview.md` §11)*

---

### 1.1 — Blocking decisions to make first

> These are the genuinely start-blocking items. Three are **Phase-A-entry blockers** (`financial-and-roadmap.md` §D2: R3 SKU freeze, R9 fail-safe ladder, instrumentation for R2/R5/R6/R8). The rest gate hardware freeze, market entry, or financing. Resolve each with the named input/experiment — do not "discover" them during diligence.

| # | Decision | Owner | Input / experiment that resolves it | What it blocks | Target by |
|---|---|---|---|---|---|
| D1 | **Freeze commercial SKU at 8× RTX 4090 (~3.3 kW nom / ~3.6 kW peak GPU).** Treat 12× as a separately-underwritten "XL" SKU. (R3) | Founders + HW eng + underwriting | Ratify 8× per `node.md` §4 reasoning (matches tape, single-tank envelope, commodity host, overlap risk). 12× = full re-underwrite, not a BOM tweak. | BOM, thermal design, the entire securitization tape & advance rate. Collateral homogeneity. | **Phase A entry / before hardware freeze** |
| D2 | **Validate true installed CAPEX ~$20–22K** (split memo's $2,500 line into compute host ~$4K + thermal BOM ~$2,500). | Underwriting + HW eng | Price a real 8-GPU host (CPU, 384 GB ECC, 8–16 TB NVMe, 2× redundant PSU, NIC, cellular, BMC, TPM) at volume; re-run payback/IRR/DSCR at $20K & $22K bookends. | Advance rate; all investor/rating materials; Slide-8 restatement. | **Before any financing (warehouse pre-read)** |
| D3 | **Secure a refurb-4090 supply channel at ~$1,800 for thousands of cards** with acceptable failure rate + warranty. | Procurement + HW eng | Source ≥2 refurb channels; sample-test failure rates; confirm 10,000+ card pipeline. Fallback: shift launch SKU to 5090 (~$2,400) → changes CAPEX & cards/kW. | The $1,800 CAPEX assumption D2 rests on; SKU economics. | **Before warehouse / scaled origination** |
| D4 | **Lock financing structure** given post-OPEX stress DSCR ~1.1–1.3× (75% advance, ~9% all-in, 5-yr amort). Decide whether to drop advance to ~70% or lean on contracted fixed legs (HSA / reserved / DR). | Finance + structuring counsel | Re-run waterfall on **net-of-OPEX** ($9,870 yr-1); note $22K @ stress (~1.14×) trips the 1.5× turbo trigger; pick advance rate + contracted-revenue floor. | Warehouse facility terms; ABS sizing; covenant headroom. | **Before warehouse close** |
| D5 | **Host agreement + transfer-on-sale clause + lien/repo strategy** drafted with counsel. Discloses compute, Superheat indemnifies, homeowner ≠ operator, term + transfer-on-sale. | Founders + legal | Counsel drafts host HSA template; define node-dark vs node-removed detection + collateral substitution rules (R12). | Any energized install; collateral integrity in the tape. | **Before first install** |
| D6 | **UL/ETL listing of the combined appliance + one reference permit** in the pilot jurisdiction. | HW eng + compliance | Engage NRTL (UL 174/1453 + IT std); NSF/ANSI 61 for wetted parts; installer pulls NEC permit + AHJ inspection (125% continuous-load rule). | Deployment in any jurisdiction; **securitization-blocking** without it (`security-and-compliance.md` §10.5). | **Phase A (before scaling installs)** |
| D7 | **Region / pilot-site selection** — 1 commercial site with controllable, near-continuous thermal load (hotel/apartment/campus). | Founders + field ops | Pick financeable commercial class (`financial-and-roadmap.md` §B1); confirm 240 V/40 A (or 208 V 3-phase), business link, permit-friendly AHJ. | Phase A entry; pilot hardware procurement; permit reference (D6). | **Phase A entry** |
| D8 | **Insurance stack bound** — product-liability + commercial policy, installer CoI, policy-boundary definition. (R13) | Finance + legal | Bind coverage ≥ required limits per SKU/region; COI on file before any install. | Any install; **securitization-blocking** if absent at diligence. | **Before first install** |

**Also confirm at hardware freeze (lower-blocking, but cheap to decide now):** modular GPU-swap requirement in the HW spec (R4, `node.md` §8); BMC-vs-agent recovery on commercial (`node.md` §9.8); cellular failover bandwidth/cost envelope (`node.md` §9.5).

---

### 1.2 — Foundational setup to start engineering

#### A. Team / roles to assign or hire (Phase A core ≈ 5–7 eng)

*(`financial-and-roadmap.md` §C2.6; roles across `operations.md`, `security-and-compliance.md`)*

- [ ] **Embedded-Linux / immutable-OS engineer** — owns the A/B image pipeline + node OS. *Done when: a named owner can build + flash a signed image.*
- [ ] **Fleet/OTA + reliability (SRE) engineer** — OTA orchestration, observability, DR runbooks. *Done when: on-call + OTA owner assigned.*
- [ ] **Networking engineer (WireGuard / NAT / CGNAT)** — overlay + relay budget. *Done when: overlay owner assigned.*
- [ ] **Firmware-adjacent thermal-controls engineer** — thermal MCU interface + safety interlock. *Done when: HIL bench owner assigned.*
- [ ] **Data/ledger engineer** — telemetry pipeline + metering tape. *Done when: tape owner assigned.*
- [ ] **Backend engineer** — registry + marketplace connector. *Done when: control-plane API owner assigned.*
- [ ] **Finance/legal/compliance liaison** (fractional ok in Phase A) — KYC/AML, host agreement, UL/permitting, insurance. *Done when: owns D4–D8.*
- [ ] *(Phase B+ add: distributed-systems/scheduler, optimization/controls, security, QA/simulator, structured-finance-systems — `financial-and-roadmap.md` §C3.6, §C4.6.)*

#### B. Repos / monorepo to create

*(`financial-and-roadmap.md` §C1.4 "build the moat, buy the boring parts")*

- [ ] **`control-plane`** — registry, telemetry ingest, scheduler (Phase B), thermal coordinator (Phase B), metering ledger/tape, renter API (Phase C), marketplace connectors. *Done when: CI builds + deploys to a staging env.*
- [ ] **`node-agent`** — the single signed `superheat-agent` daemon (control link, telemetry, OTA, job runtime, thermal/power controller, watchdog, attestation). *Done when: agent builds + passes `agentctl ping` on the bench.*
- [ ] **`node-os`** — Flatcar-derived immutable image build (NVIDIA driver + Container Toolkit baked, dm-verity, A/B partitions, health-gate.yaml). *Done when: a signed A/B image boots on real hardware.*
- [ ] **`infra-as-code`** — Terraform/Pulumi for cloud, Postgres/Supabase, TSDB, relay/headscale, Grafana, secrets. *Done when: `apply` stands up the staging stack from scratch.*
- [ ] **`data-contracts`** — the NORMATIVE wire schemas (`superheat.telemetry.v1`, `superheat.usage.v1`, etc.) as the shared dependency every producer round-trips. *Done when: schema package is versioned + consumed by agent + control-plane.*

> Suggested layout: a monorepo (`/superheat`) with `control-plane/`, `node-agent/`, `node-os/`, `infra/`, `contracts/`, `docs/` (the 8 design docs). `contracts/` is the single source of truth; CI fails any producer that drifts from it (`operations.md` §8.3 contract conformance).

#### C. Tech-stack picks (the design already implies these — adopt, don't re-debate)

| Layer | Pick | Build/Buy | Source |
|---|---|---|---|
| Node OS | **Flatcar-style immutable A/B** (Ubuntu Core / Talos = fallbacks) | Buy base, build image | `node.md` §11.3 |
| Overlay network | **WireGuard mesh — self-hosted headscale** (or Tailscale), outbound-only, relay/TURN fallback for CGNAT | Buy/self-host | `connectivity-and-data.md` §1; `node.md` §0 |
| Container runtime | **NVIDIA Container Toolkit** (hardened containers; microVM/VFIO for commercial multi-tenant) | Buy | `platform.md` §3.2–3.3 |
| Registry + ledger DB | **Managed Postgres (Supabase)** — one transactional store for registry + tape | Buy | `platform.md` §1.1, §7.1 |
| Hot node-state cache | **Redis** | Buy | `platform.md` §5 |
| Telemetry ingest | **Kafka / Redpanda** | Buy | `platform.md` §2 |
| Time-series store | **VictoriaMetrics or Timescale** (hot 30d / warm 13mo / cold Parquet 5y) | Buy | `platform.md` §2.3; `operations.md` Part A §2 |
| Dashboards/alerting | **Grafana** | Buy | `platform.md` §2; `operations.md` §2.3 |
| Fleet/OTA substrate | **Mender / Balena** (distribution + A/B); build the staged-rollout policy engine | Buy substrate, build policy | `platform.md` §3 |
| Payments/settlement | **Stripe Connect** (+ ACH) with OFAC screening | Buy | `platform.md` §7.2; `security-and-compliance.md` §11.2 |
| Agent language | **Rust or Go** (single static signed daemon, hardware-near) | Build | `node.md` §13 |
| Services language | **Go / TypeScript** (backend) + **Python** (thermal/forecast models) | Build | `platform.md` §5–6 |
| **Build (moat):** scheduler, thermal coordinator, yield router, metering ledger, staged-rollout/health-gate, edge aggregator. | | Build | `financial-and-roadmap.md` §C1.4 |

- [ ] Ratify the table above in a one-page ADR. *Done when: each row has a named owner and is reflected in `infra-as-code`.*

#### D. External accounts / vendors to set up

- [ ] **Cloud account** (control-plane host) + org/billing + IaC service principal. *Done when: `infra-as-code apply` runs.*
- [ ] **vast.ai host account** (Phase-A revenue attach; first marketplace, then RunPod/Salad/io.net/Akash in B/C). *Done when: a test node lists on vast.ai.* (`platform.md`)
- [ ] **Supabase project** (managed Postgres = registry + ledger). *Done when: schema migrated from `data-contracts`.*
- [ ] **headscale host** (self-hosted WireGuard coordination) + relay/DERP POP. *Done when: a node dials out + reaches the control plane, zero inbound ports.* (`connectivity-and-data.md`)
- [ ] **Cellular SIM provider** (commercial LTE/5G failover, metered as OPEX). *Done when: one SIM + auto-failover router provisioned for the pilot.* (`node.md` §6.2)
- [ ] **Stripe Connect** account + OFAC/sanctions screening flow. *Done when: a test payout clears with KYC.* (`security-and-compliance.md` §11.2 — **securitization-blocking** without it)
- [ ] **Observability stack** (Grafana Cloud or self-hosted + VictoriaMetrics/Timescale). *Done when: the canonical telemetry frame renders on a fleet dashboard.* (`operations.md` §2.3)
- [ ] **Image-signing / secrets** (KMS or equivalent) for signed OS images + node identity keys (TPM-backed). *Done when: an image won't boot unsigned.* (`node.md` §18)

#### E. Pilot — 1 commercial site + hardware + bench

- [ ] **Pick 1 commercial pilot site** with controllable, near-continuous thermal load (per D7). *Done when: site + host agreement + permit-friendly AHJ confirmed.*
- [ ] **Procure pilot hardware: 1× commercial 8× refurb-4090 host** per `node.md` §5.2 BOM (8× 4090, ≥48-core CPU, 384 GB ECC, 8–16 TB NVMe, 2× redundant PSU, NIC + cellular, TPM, BMC, thermal MCU). *Done when: host assembled + powers on.*
- [ ] **Stand up the bench / HIL rig** per `operations.md` §12.1: real 4090 + real thermal MCU (firmware) + immutable OS + agent + real NIC; thermal emulator (real tank or validated heat-load + simulated tank model); reverse-tunnel console + control-plane stub; fault injectors (sensor faults, over-temp, comfort draws, link loss, breaker excursions). *Done when: the rig can run the §12.2 safety suite.*
- [ ] **Run the safety-blocking HIL suite** (`operations.md` §12.2): fail-safe-to-heater (kill agent/OS/both USR slots → MCU keeps heating), `heater_safe=false` interlock (HALT_GPU within 1 tick), thermal-link loss, over-temp/breaker (HALT or THROTTLE, never trip breaker), A/B health-gate rollback. *Done when: S1 (zero-hot-water) and S14 (OTA rollback ≥99.5% self-recovery) gates pass — these are R9 release gates.*
- [ ] **Field-test a GPU module swap + version-aware re-registration** on the pilot unit (R4). *Done when: swapped card auto-detected + re-registered at new generation.* (`node.md` §8.2)

---

#### Phase-A "ready to build" gate (when Part 1 + Part 2 are done)

- [ ] D1 (SKU frozen 8×), D6 (UL + reference permit underway), D7 (pilot site picked) — the three Phase-A-entry blockers.
- [ ] Repos B + contracts schema in place; tech-stack ADR ratified; vendor accounts D live.
- [ ] Pilot host built; HIL rig passing S1 + S14.
- [ ] Telemetry + minimal registry + metering-v0 reading per-unit GPU-hours into the tape (the measurement program that quantifies R2/R5/R6/R8).

*Exit target for Phase A (for reference, not this step): ≥1,000 commercial units, 12 months clean per-unit tape, no single marketplace >60%. (`financial-and-roadmap.md` §C2.4)*

---
*Cross-references: `system-design-overview.md` (principles, phasing, open questions) · `node.md` (SKU freeze, BOM, OS/agent, thermal interface, GPU swap, TPM) · `connectivity-and-data.md` (overlay, relay/cellular, cache) · `platform.md` (control plane, build-vs-buy, marketplace attach, tape) · `security-and-compliance.md` (UL/permitting, host agreement, KYC, insurance, privacy) · `financial-and-roadmap.md` (CAPEX, OPEX/DSCR, roadmap C1–C5, risks R1–R15) · `operations.md` (observability, testing, HIL bench, field ops) · `data-contracts.md` (normative schemas).*

---

## Part 2 — Phase A: Attach for revenue

**Status:** Build playbook · **Phase:** A (asset-proving / tape-building) · **Audience:** engineering leads, SRE, on-call, field-ops, underwriting liaison

**What this part is.** An ordered, do-able build sequence for Phase A of Superheat. Each step gives the **action** (what to build, what tech), **prerequisites**, **done-when** (acceptance), and the **design § to follow**. Rationale lives in the design docs — this is the checklist, not the argument.

**Phase A goal (`financial-and-roadmap.md` §C2).** Get nodes earning on existing marketplaces (vast.ai) with safe remote ops, producing the clean per-unit metered tape that Phase-1 financing (equity pilot → venture debt, ladder rungs 0–1) underwrites. Phase A is as much a *measurement* program as a revenue program: it converts the overview's open questions (R2 relay cost, R5 seasonality, R6 concentration, R8 tape integrity) into quantified, underwritable numbers (`financial-and-roadmap.md` §D2).

**The five invariants (hold from the first node, never a phase — `financial-and-roadmap.md` §C1.3):** (1) outbound-only; (2) fail-safe-to-heater; (3) thermally-coupled compute; (4) measure-then-price reliability; (5) hybrid demand.

**Hard gates.** Steps tagged **⛔ GATE** are safety-critical fail-safe-to-heater work. They **block fleet rollout**: no OTA reaches canary, and no rollout advances to fleet, until the gate is green (`operations.md` §13.2, R9). Gates are listed in workstream 6.

**Build vs buy (`platform.md` §10).** Buy the boring substrate — Postgres/Supabase, TSDB, Kafka/Redpanda, OTA transport, payout rails, headscale/DERP. Build the four moat pieces — but in Phase A only the *skeleton* of the control plane and the *read-only* tape.

---

### Workstream entry gate (resolve before any Phase A build)

*(`financial-and-roadmap.md` §C2.3, §D2)*

- [ ] **E1 — Freeze the commercial SKU at 8× RTX 4090 (`COMM-4090x8-v1`).** Ratify before hardware freeze; resolves start-blocking risk **R3**. Locks BOM, thermal envelope (~3.3 kW nominal / ~3.6 kW design point), and the tape's collateral definition. **Done when:** SKU row exists in `sku` table with `gpu_count=8`, `gpu_tdp_w=412`, `usable_thermal_kwh` set per-SKU. **§:** `node.md` §4, `financial-and-roadmap.md` §A6.
- [ ] **E2 — Secure ≥1 pilot site with controllable, near-continuous thermal load** (commercial host preferred — the financeable class). **Done when:** site assessment passes (`operations.md` §20.5.1) and host agreement with transfer-on-sale clause is signed. **§:** `financial-and-roadmap.md` §B1, `operations.md` §20.5.
- [ ] **E3 — Equity pilot capital committed** (ladder rung 0, ~$2–5M). **Done when:** committed, so deployment run-rate never exceeds committed capacity (R10). **§:** `financial-and-roadmap.md` §C2.7.
- [ ] **E4 — Validate two procurement assumptions:** sustainable refurb-4090 channel at ~$1,800 for pilot volume, and the real 8-GPU host cost. **Done when:** signed procurement quotes exist; CAPEX restated to ~$20–22K. **§:** `financial-and-roadmap.md` §A2, `node.md` §9.

---

### Workstream 1 — Node OS + Agent

> Builds: immutable OS with A/B + rollback, node agent v1, thermal guard, tenant isolation, attestation. Owner: embedded-Linux + fleet/OTA + thermal-controls engineers.

#### 1. Build the immutable A/B OS image
- **Action:** Stand up a **Flatcar-derived immutable image** — read-only dm-verity root, whole-image A/B (USR-A / USR-B slots), no package manager, signed images only. Bake in the **pinned NVIDIA driver + CUDA runtime + NVIDIA Container Toolkit** (sysext/overlay layer) and the partition layout (ESP / USR-A / USR-B / OEM / STATE / CACHE / SCRATCH). Steady-state has **no interactive shell** (Talos idea borrowed).
- **Prereqs:** E1 (SKU freeze fixes GPU count / driver target).
- **Done when:** a node boots a signed image to a verified read-only root; `STATE`/`CACHE`/`SCRATCH` survive an image change; a water-heater-safe minimal init exists in *both* USR slots.
- **§:** `node.md` §11 (OS), §11.4 (partitions).

#### 2. Build the CI image pipeline (the only way software reaches a node)
- **Action:** CI builds the image → pins/records every version → emits a single **content-addressed image digest** (the unit of attestation and rollout). Sign image; publish to the release set the control plane draws from. Driver/CUDA upgrades are *image* events, never in-place installs.
- **Prereqs:** Step 1.
- **Done when:** a commit produces a reproducible signed digest; the digest is the only artifact the OTA orchestrator (Step 18) can target.
- **§:** `node.md` §17.

#### 3. Build node agent v1 (`superheat-agent`)
- **Action:** Single signed privileged daemon with the module set in `node.md` §13.2: `ControlLink` (outbound mTLS/gRPC over overlay), `Supervisor` (state machine), `TelemetryCollector`, `JobRuntime`, `OTAMgr`, `ThermalPowerController` (local safety authority), `HeaterIface`, `WatchdogMgr`, `StateStore` (SQLite/BoltDB on `STATE`). Implements the state machine `PROVISIONING→IDLE→RUNNING-JOB→DRAINING/PREEMPT→UPDATING→RECOVERING→HEATER-ONLY SAFE-MODE`.
- **Prereqs:** Step 1.
- **Done when:** agent runs as the only privileged long-running process; `agentctl ping` responds; state transitions are logged to `StateStore`; `HEATER-ONLY SAFE-MODE` is reachable from every state.
- **§:** `node.md` §13, §14.4.

#### 4. Build the A/B update + automatic rollback machinery
- **Action:** Implement the boot flow with **health gate** (`/etc/superheat/health-gate.yaml`, signed, in-image): boot tentative slot with try-counter (MAX=3); agent runs the gate then `bootloader_commit()` on pass. Two rollback triggers: **hard** (bootloader auto-rollback if agent never commits within MAX attempts — needs no working agent) and **soft** (agent fails a *heater-critical* check → reboot toward rollback). A *compute-optional* failure → `commit_degraded` into heater-only, **never** a rollback loop. Add the **post-commit health-regression watchdog** (24 h soak window; auto-revert on crash-loops / XID storms / thermal-link flap = `reason=post_commit_regression`).
- **Prereqs:** Steps 1, 3.
- **Done when:** a bad-driver image fails the gate and auto-rolls-back to last-good; a dead-GPU-but-heats image commits degraded and does NOT loop; agent emits `ota_result {from_digest,to_digest,gate_results[],committed|rolled_back|committed_degraded,duration_s}`.
- **§:** `node.md` §12 (all subsections).

#### 5. Build the thermal guard (local safety only — NOT the optimizing coordinator)
- **Action:** Implement the **local control loop** (1 Hz tick) in `ThermalPowerController` — the only path that can change GPU power state. Order: **hard safety halts first** (gpu_temp>CRIT, tank_temp>MAX, `heater_safe=false`, thermal link down, no coolant flow, lost remote heartbeat → `HALT_GPU`), then **power throttle** (near `CIRCUIT_BUDGET_W` with hysteresis — clamp, never halt), then **thermal-coupling** (no storage headroom + no demand → halt), then remote plan. Use `derive_cap()` to power-cap the GPU (412–450 W envelope) so heat-out ≈ tank-absorbable now. Import per-SKU `usable_thermal_kwh` / `η_capture` from the registry — **never hardcode `12.0`**.
- **Prereqs:** Steps 3, 11 (HeaterIface contract).
- **Done when:** local safety always wins over any remote plan; near-breaker conditions only clamp; `CIRCUIT_BUDGET_W` is SKU-derived, not a literal; storage-headroom predicate matches the telemetry `storage_headroom_kwh` formula.
- **§:** `node.md` §14.3, §7.6; `data-contracts.md` §6.

#### 6. Build tenant isolation + egress firewall
- **Action:** Default tenant runtime = OCI containers via NVIDIA Container Toolkit, rootless/locked-down (dropped caps, seccomp, no host PID/IPC/net, no path to HeaterIface socket or agent API). GPU power cap set *outside* the container. Implement the **per-tenant egress firewall**: default DENY; allow only regional cache + tenant allowlist; **deny-always** the home LAN (RFC1918), overlay control subnet, heater MCU, mining/abuse lists; log all connections. (microVM/Cloud-Hypervisor+VFIO is a Phase-C trusted-host concern — not Phase A.)
- **Prereqs:** Steps 3, 13 (overlay).
- **Done when:** a tenant container cannot reach `192.168.x.0/24`, the agent socket, or the MCU; all egress is logged; GPU power cap is not tenant-raisable.
- **§:** `node.md` §14.5, §14.7.

#### 7. Build attestation + identity (software baseline + TPM where present)
- **Action:** Each node generates a **hardware-bound key in TPM/secure element** at provisioning; public key registered to `node.attestation_pk`. First boot = `PROVISIONING` with a one-time short-lived enrollment token. Implement **attest-before-paid-work**: control plane challenge(nonce) → agent signs quote {image_digest, agent measurement, secure-boot/dm-verity state, thermal-link present} → control plane verifies digest ∈ allowed_release_set + sig chains to registered key. Track assurance class: TPM-equipped = standard; **software-only = lower-assurance, flagged on the tape** (`trusted_host`-ineligible).
- **Prereqs:** Steps 3, 17 (registry).
- **Done when:** only attested image+identity node-hours can be scheduled paid work; a node that fails attestation still heats but earns nothing; re-attest fires on every post-OTA commit.
- **§:** `node.md` §18; `connectivity-and-data.md` §A8; `security-and-compliance.md`.

#### 8. Implement the agent's telemetry + usage-record producers
- **Action:** `TelemetryCollector` emits the canonical **`superheat.telemetry.v1`** frame (`data-contracts.md` §5) outbound-only on a cadence (~10–30 s heartbeat, richer on state change): `gpus[]` array (length 1 H1 / N commercial), `mem_used_mb`, `thermal.mode` enum + `thermal.heat_demand_w` watts, `board_power_w`, `net.path ∈ {direct,relay}`, `cloud_degraded`, monotonic `seq`, `sig` over the node key. Agent also emits the signed **`superheat.usage.v1`** record (`data-contracts.md` §8) per (gpu_slot, job, period) with `node_sig` — per-GPU, never spanning a period boundary.
- **Prereqs:** Steps 3, 11 (thermal data), 7 (signing key).
- **Done when:** both objects round-trip the `data-contracts.md` schemas with zero field loss (conformance test, Step 30); `gpus` is always an array; usage records carry mandatory `gpu_slot`.
- **§:** `node.md` §15; `data-contracts.md` §5, §8.

---

### Workstream 2 — Connectivity

> Builds: WireGuard outbound-only overlay + relay fallback + NAT probe + one regional PoP cache. Owner: networking engineer.

#### 9. Stand up the overlay coordination plane
- **Action:** Deploy **self-hosted headscale** as the coordination server in our cloud (next to the control plane), with the OSS WireGuard/Tailscale client on nodes and **self-hosted DERP relays** (one POP for the pilot region). Use **Tailscale SaaS for the internal operator tailnet** to move fast. Keep the agent's tunnel behind an internal `OverlayProvider` interface (Nebula is the documented fallback). Dual-stack (v4+v6) everywhere.
- **Prereqs:** E2 (pilot region known).
- **Done when:** a node dials out, authenticates (signed pre-auth key + attestation), gets a stable overlay IP on the control subnet (`100.64.0.0/16`), and is reachable by control plane + operators — **with zero inbound ports on the home router**.
- **§:** `connectivity-and-data.md` §A2, §A3.2, §A5.

#### 10. Implement outbound-only dial + NAT traversal + relay fallback + NAT probe
- **Action:** `persistent-keepalive ~25 s`; **IPv6-first** candidate selection (highest-leverage CGNAT mitigation); STUN + hole-punch with port-prediction; **DERP relay fallback** when direct fails (prefer-direct with continuous background upgrade). On boot, run a **NAT-type probe** (full-cone/restricted/port-restricted/symmetric/CGNAT-hard) and report it + IPv6 capability to the control plane. Treat connectivity loss as fail-safe: node degrades to ordinary water heater, compute disabled.
- **Prereqs:** Step 9.
- **Done when:** nodes behind CGNAT still connect (via relay); `net.path` (direct|relay) and NAT classification are reported per node; existing sessions survive a coordinator blip (cached peer config).
- **§:** `connectivity-and-data.md` §A4, §A9, §A11.

#### 11. Build the HeaterIface ↔ thermal MCU link + message contract
- **Action:** Implement the agent↔MCU bus (UART/CAN/I²C) with the §16.2.1 message contract. Agent→MCU: `gpu_heat_intent`, `request_window`, `set_pump`, `agent_alive`. MCU→Agent: `thermal_status`, `heater_safe` (master safety signal), `comfort_constraint`, `fault`. Safety semantics: `heater_safe=false` ⇒ immediate non-overridable `HALT_GPU`; **loss of link = unsafe = HALT_GPU**; MCU's own watchdog forces conventional-heating-only if `agent_alive` stops. MCU firmware updates on a **separate cadence/slot** from the host OS.
- **Prereqs:** Step 3; thermal-controller hardware (Step relies on `node.md` §7 BOM).
- **Done when:** the MCU heats water entirely on its own with the agent dead; `heater_safe=false` halts GPU within one control-loop tick; both sides fail safe toward "just a water heater" on link loss.
- **§:** `node.md` §16.2; §7.3.

#### 12. Stand up one regional PoP cache (content-addressed) + on-node NVMe cache
- **Action:** Deploy **one regional content-addressed PoP cache** (`cas:sha256:<digest>` blobs, 10–100 TB NVMe, 1–10 Gbps backhaul). On-node: LRU + pinning cache on the `CACHE` partition with a reserved checkpoint-scratch slice. Jobs pull base images/weights *down* from the PoP (fast direction), never over the renter's uplink. (Bulk-off-relay collapses the relay cost line — the key R2 lever.) Mandatory checkpoint/resume SDK is Phase B; Phase A just needs local NVMe scratch + cache pulls.
- **Prereqs:** Steps 9, 10.
- **Done when:** a job's base image/weights are served from the PoP/on-node cache; blobs are hash-verified on arrival; relay-bytes vs direct-bytes is measured (feeds R2).
- **§:** `connectivity-and-data.md` §B1, §C1 (Phase A row).

---

### Workstream 3 — Control-plane skeleton

> Builds: minimal registry, telemetry pipeline, OTA orchestrator, reliability scoring v0. Owner: backend + data + SRE. Buy the substrate, build the thin APIs.

#### 13. Stand up the registry (Postgres/Supabase)
- **Action:** Buy managed Postgres (Supabase). Create the schema: `owner`, `installer`, `sku`, `node` (with `lifecycle_state`, `tranche_id` financing tag, `attestation_pk`, `agent_version`, `os_slot`, `enrolled_at`), `gpu` (per-GPU row, `gpu_one_live_per_slot` partial unique index so a year-5 swap reuses the slot), `node_lifecycle_event` (append-only audit). Build the thin `RegistryService` (`GetNode`, `ListNodes`, `UpdateLifecycle`, `AssignTranche`, `RegisterGpuSwap`) + the lifecycle FSM (`provisioned→installed→attested→active⇄degraded→…`). Redis read-cache (30 s TTL).
- **Prereqs:** E1.
- **Done when:** every node/GPU is registered with SKU, region, owner, and **financing tranche tag from day one**; lifecycle changes append an event; only `attested` nodes can be assigned paid work.
- **§:** `platform.md` §1.

#### 14. Build the telemetry pipeline (edge aggregator → ingest → TSDB)
- **Action:** Buy TSDB (VictoriaMetrics/Timescale/Mimir) + Kafka/Redpanda. Build the **regional edge aggregator**: terminates node overlay streams, validates the attestation token on each frame, **decimates 10 s → 1 min** for cloud, keeps 10 s locally 24 h, and **buffers ≥2 h** so a cloud-ingest outage causes zero telemetry loss (nodes never block on cloud). Ingest → TSDB hot tier (1-min rollups, 30 d) + scoring consumer + alert evaluator.
- **Prereqs:** Steps 8, 9, 13.
- **Done when:** frames flow node→edge→cloud; ingest availability ≥99.9% (S2), visibility p95 <60 s (S3); aggregator buffers during a cloud blip with zero loss (S19).
- **§:** `platform.md` §2; `operations.md` §1 (S2/S3/S19), §2.2 (cardinality rules).

#### 15. Build reliability scoring v0 + seasoning gate
- **Action:** Implement `reliability_score(...)` per `data-contracts.md` §3 — score ∈ [0,100], positive weights sum to 1.0 (**perfect node == 100**), fault penalty caps at −30%, **`cloud_degraded` windows excluded from uptime** (the false-outage fix). Apply the **seasoning gate**: <30 days clean history ⇒ `probation` and **not tape-eligible** regardless of score. Persist `node_reliability_history` with `seasoned` flag. Tiers: ≥85 premium / ≥70 standard / ≥50 spot / else probation.
- **Prereqs:** Step 14.
- **Done when:** scores recompute ≥ daily/node; `cloud_degraded` gaps never demote a node; unseasoned nodes are `probation`/`seasoned=false` and excluded from the tape (Step 23).
- **§:** `platform.md` §4; `data-contracts.md` §2, §3.

#### 16. Build the cloud-side attestation verifier + counter-signer
- **Action:** Implement the control-plane half of Step 7: verify attestation quotes, gate `lifecycle_state→attested`, and **counter-sign usage records** (`cloud_sig`) on ingest after reconciliation — producing the double-signed provenance chain. Reject unsigned / single-signed / replayed (duplicate `usage_id`) / period-spanning records at counter-sign.
- **Prereqs:** Steps 7, 13.
- **Done when:** only genuine image+identity nodes reach `attested`; every accepted usage record is double-signed; tape-poisoning attempts are rejected before entering `usage_record`.
- **§:** `platform.md` §7.1; `node.md` §18.2; `data-contracts.md` §8.

#### 17. Build the provisioning/enrollment + installer commissioning flow
- **Action:** Factory/warehouse stages a node in `provisioned` with TPM public key registered and a one-time enrollment token bound to that node record. First boot: agent dials out, presents `{node_id, enrollment_token, hw_key_pub, sign(nonce)}`, control plane verifies + burns the token + issues overlay creds. Build the **commissioning app** (thin control-plane client on the installer's phone — talks only to the control plane, never to the node, **cannot** read keys / set `attested` / touch router config). Acceptance suite: heater-only proof, thermal link + sensors, overlay reachability, GPU smoke test, attestation.
- **Prereqs:** Steps 7, 9, 13, 16.
- **Done when:** an installer brings a node to `installed` and the control plane flips it to `attested` cryptographically — **without the installer ever touching the customer's router**; a malicious installer's worst case is a *failed* commission, never a forged one.
- **§:** `operations.md` §21, §20.5.

#### 18. Build the OTA orchestrator (staged rollout + health-gate)
- **Action:** Buy OTA artifact/A-B transport (Mender/Balena-style or OS-vendor server). Build the Superheat-specific policy: `ota_release` / `ota_rollout` (`stage ∈ canary|cohort|fleet|halted|complete|rolled_back`, `cohort_filter`) / `ota_node_status`. Staged flow **canary (1%, biased to low-reliability + idle/heating-only nodes, never a non-interruptible reserved node, hold ≥6 h) → cohort (5→25→50%, one region/wave, hold ≥2 h) → fleet (capped %/hr)**. Health-gate promotion vs a held-back control group: boot-success ≥99.5%, crash ≤1.5× baseline, GPU-fault not elevated, **thermal-safety faults = 0 (hard stop)**. **Auto-halt <10 min + auto-rollback + quarantine** on breach. Nodes only ever pull the digest assigned to their cohort + apply on their own thermal/job-aware schedule.
- **Prereqs:** Steps 2, 4, 13, 14.
- **Done when:** a clean release walks canary→fleet in <14 days (slow by design, S15); a bad rollout auto-halts <10 min (S13) and self-heals via node-side A/B; rollback success ≥99.5% with OTA-attributable thermal faults = 0 (S14).
- **§:** `platform.md` §3; `node.md` §12.3.

---

### Workstream 4 — Marketplace Attach

> Builds: vast.ai connector + single-source router + concentration *measurement*. Owner: backend engineer.

#### 19. Implement the canonical `superheat.job.v1` schema + connector interface
- **Action:** Implement `superheat.job.v1` (`data-contracts.md` §4) and the `MarketplaceConnector` Protocol (`list_offer`, `claim`, `normalize`, `report_usage`, `fetch_settlement`). `source ∈ {vast,runpod,salad,ionet,akash,direct,reserved,backlog}`; durations in **seconds**; `gpu_model == "RTX4090"` exact token; `net = gross·(1−fee)`.
- **Prereqs:** Step 13.
- **Done when:** the normalized `Job` type-checks across the connector→scheduler boundary (conformance test, Step 30).
- **§:** `platform.md` §13; `data-contracts.md` §4.

#### 20. Build the vast.ai connector (Phase A ships vast.ai *only*)
- **Action:** Implement the vast.ai `MarketplaceConnector`: run the `vastai` host daemon inside the agent's isolated container path; list the GPU at an ask price; renter rents → daemon pulls renter container into the locked-down runtime (Step 6). Capture delivery telemetry → metering. vast.ai is the single fastest path to revenue and the deepest 4090 pool, so it ships first and alone.
- **Prereqs:** Steps 6, 8, 19.
- **Done when:** a real vast.ai renter job runs in an isolated container on a pilot node, inside a thermal window, and produces signed usage records.
- **§:** `platform.md` §12, §19 (Phase A row).

#### 21. Build the trivial single-source router + concentration *measurement*
- **Action:** Trivial Phase-A router = "always vast.ai." But **instrument per-marketplace revenue share from day one** (trailing-30-day `mix[source]` from the ledger) and report the **marketplace-concentration metric**. The hard 60% guard / multi-source steering is Phase B — Phase A only needs to *measure and report* the metric (it will read 100% vast.ai, which is the point: prove the metric works and is <60% once Phase B diversifies).
- **Prereqs:** Steps 20, 23 (ledger feeds the share).
- **Done when:** a dashboard reports % revenue per marketplace (R6); the metric is wired to the exit-criteria report.
- **§:** `platform.md` §16, §19; `financial-and-roadmap.md` §C2.4 (R6).

---

### Workstream 5 — Metering v0 (read-only tape)

> Builds: the append-only signed-usage ledger + the read-only securitization tape + manual reconciliation v0. **The tape is the product.** Owner: finance-systems + ledger eng. Must be correct from day one (R8).

#### 22. Build the append-only usage ledger
- **Action:** Create `usage_record` (immutable; per-GPU via `node_id + gpu_slot`, FK to the exact `gpu` row so it survives slot reuse/year-5 swap; `node_sig` + `cloud_sig`; never spans a period boundary), `settlement` (incl. `electricity_offset_usd`, `marketplace_payout_ref`), `tranche`. Indexes on `(node_id, gpu_slot, start_ts)` and `(tranche_id, start_ts)`. Ledger lives in the **same Postgres as the registry** so settlement and identity are transactional together.
- **Prereqs:** Steps 13, 16.
- **Done when:** 100% of paid GPU-hours are backed by a double-signed `usage_record` (S10); the ledger is append-only and reproducible.
- **§:** `platform.md` §7.1, §7.2.

#### 23. Build the read-only securitization tape (materialized view)
- **Action:** Build the `securitization_tape` materialized view: aggregate **per-GPU first** (join `usage_record` to `gpu` on **both** `node_id` AND `gpu_slot` — fixes the Cartesian-multiply bug), then roll up to node-per-period. Filter `lifecycle_state='active' AND seasoned=true`. Surface per node/period: GPU-hours, gross, platform fee (25%), realized $/GPU-hr, utilization %, uptime, thermal availability, reliability score/tier. **Read-only / "reported by the marketplace, reconciled against telemetry"** — authoritative billing is Phase B; Phase A proves the tape is *capturable and auditable*.
- **Prereqs:** Steps 15, 22.
- **Done when:** the tape produces one clean row per active+seasoned node per period; re-running the view over the append-only table reproduces an identical tape (re-runnable = audit guarantee); monthly close <24 h (S12).
- **§:** `platform.md` §7.3; `financial-and-roadmap.md` §C2.4.

#### 24. Build settlement reconciliation v0 (manual variance review)
- **Action:** Reconcile **signed usage records vs the vast.ai settlement statement** in USD (a *money* check, never a telemetry check). Phase A = manual variance review; flag drift. Buy Stripe Connect / ACH payout rails for owner settlement. (Automated three-way reconciliation incl. bank cash is the Phase-B exit gate for R8.)
- **Prereqs:** Steps 22, 23.
- **Done when:** every period's tape revenue reconciles to vast.ai's actual payout within tolerance; variances produce a written report; the ±tolerance is applied to money, not telemetry.
- **§:** `platform.md` §18; `operations.md` §14.3; `data-contracts.md` §8.

---

### Workstream 6 — Safety / Release-gating (HARD GATES — block fleet rollout)

> These are invariants from the first node (R9), not a phase. Owner: node-software + thermal-controls + SRE. **No OTA reaches canary, and no rollout advances to fleet, until every gate is green** (`operations.md` §13.2).

#### 25. ⛔ GATE — Build the HIL safety bench (≥1 rig per hardware variant)
- **Action:** Build a hardware-in-the-loop bench running the **exact** field image + agent: real node host + immutable OS + `superheat-agent`, **real GPU (4090)**, **thermal MCU on real firmware** (it is the local safety authority — must be real silicon, not mocked), a thermal emulator (real tank or validated electrical heat-load + tank model) that injects sensor faults / over-temp / comfort draw / link loss / breaker excursions, and a reverse-tunnel console + control-plane stub.
- **Prereqs:** Steps 1, 3, 4, 5, 11.
- **Done when:** the bench runs the field image unmodified; `η_capture` and `C_th` are **measured on this rig** to replace the `data-contracts.md` §6 placeholders.
- **§:** `operations.md` §12.1; `node.md` §17.

#### 26. ⛔ GATE — Pass the HIL safety-critical suite (BLOCKING before ANY OTA)
- **Action:** Run and pass the §12.2 blocking suite: **fail-safe-to-heater** (kill agent / kill OS / corrupt both USR slots → MCU keeps heating); **`heater_safe=false` interlock** (HALT_GPU within one tick); **thermal-link loss**; **over-temp / breaker** (halt vs throttle, never trips breaker); **A/B health gate** (broken driver → no commit → auto-rollback); **A/B hard rollback** (image hangs pre-agent → bootloader rollback after MAX=3); **GPU-broken→heater-safe** (degrade, don't loop); **recovery ladder** all rungs 0→4; **attestation post-OTA**; **watchdog** (hung loop → reboot).
- **Prereqs:** Step 25.
- **Done when:** every row passes; a failure halts the release pipeline.
- **§:** `operations.md` §12.2; `node.md` §16.1 (recovery ladder).

#### 27. ⛔ GATE — Prove A/B rollback in production (canary → cohort → fleet, zero hot-water incidents)
- **Action:** Wire the HIL/CI gates into the OTA staged rollout (Step 18): CI green → sim-smoke → **HIL safety suite green (blocking)** → staging E2E+chaos → canary 1% (≥6 h) → cohort waves (≥2 h/wave) → fleet, with auto-halt/rollback/quarantine on any breach. Run a real rollout on the pilot fleet and observe an **auto-rollback in production** with the recovery ladder firing.
- **Prereqs:** Steps 18, 26 (HIL suite green).
- **Done when:** rollback is demonstrated in production through canary→cohort→fleet with **zero customer-hot-water incidents** (S1=0, S14 thermal-faults=0); rollback success ≥99.5%.
- **§:** `operations.md` §13.1; `financial-and-roadmap.md` §C2.4.

#### 28. ⛔ GATE — Pass the zero-hot-water-incident release-gate checklist
- **Action:** Enforce the `operations.md` §13.2 checklist as a standing gate on every release: unit math green (score==100, `fits()`, period-split, no `12.0` literal); contract-conformance green (Step 30); sim comfort-violations=0; **HIL safety suite green (blocking)**; chaos degrades gracefully + auto-halt <10 min; tape correctness green (Step 29); canary excludes non-interruptible reserved nodes; rollback rehearsed on the bench for this exact digest; on-call + runbook ready; OTA-attributable thermal-fault count across canary+cohort = 0 before fleet.
- **Prereqs:** Steps 26, 27, 29, 30.
- **Done when:** a single failed box halts the release; the gate is the operational form of R9/S1.
- **§:** `operations.md` §13.2.

#### 29. Build tape-correctness + chaos tests
- **Action:** **Tape property tests** (`operations.md` §14.1): double-signing required; per-GPU join (no Cartesian multiply); period-boundary split; conservation (`gross == fee + net`); **telemetry is never the tape**; idempotent tape rebuild. **Golden tests** (§14.2) incl. month-boundary, multi-GPU node, GPU module-swap mid-period, probation node excluded. **Chaos / fault-injection** (§16) on the recovery ladder: correlated cloud/aggregator outage (nodes keep heating, buffer ≥2 h, `cloud_degraded` stamped, uptime not penalized); bad OTA; GPU XID storm; preemption (`T_GRACE_S=30`).
- **Prereqs:** Steps 22, 23, 14.
- **Done when:** tape invariants hold over generated inputs; tape-poisoning rejected; correlated-outage chaos proves the `cloud_degraded` exclusion (protects R8 + S16).
- **§:** `operations.md` §14, §16.

#### 30. Build the canonical-contract conformance suite (NORMATIVE, CI-blocking)
- **Action:** Every producer/connector must **round-trip** the `data-contracts.md` schemas (`job.v1`, `telemetry.v1`, reliability score, `usage.v1`, thermal constants, `fits()`) — serialize→deserialize→re-validate, zero field loss, zero illegal values. Maintain a versioned **golden corpus** (valid + adversarial); a `data-contracts.md` change that doesn't update the corpus fails CI.
- **Prereqs:** Steps 8, 19.
- **Done when:** divergence from the canonical schemas is a build failure; `RTX4090` exact-token, `gpus`-always-array, mandatory `gpu_slot`, and "no `12.0` literal" are all asserted.
- **§:** `operations.md` §8.3; `data-contracts.md` (all NORMATIVE §).

---

### Phase A Exit Checklist (= Phase-1 financing entry)

*(`financial-and-roadmap.md` §C2.4)*

Each exit criterion maps to the step(s) that produce and verify it.

- [ ] **≥1,000 commercial units deployed and instrumented.**
  *Verified by:* Steps 17 (provisioning/commissioning at scale), 13 (registry inventory), 14 (telemetry instrumentation per node). *Metric:* count of `active` nodes in the registry. *§:* `platform.md` §1; `operations.md` §20.

- [ ] **12 consecutive months of clean per-unit monthly tape** (metered GPU-hours, realized $/hr, utilization, uptime, thermal draw, host payment behavior).
  *Verified by:* Steps 22 (signed ledger), 23 (read-only tape, S12 monthly close), 24 (reconciliation v0), 15 (seasoning so only seasoned nodes are on-tape), 29/30 (tape correctness + conformance). *Metric:* S10 100% coverage, S12 <24 h close, reconciliation variance within tolerance for 12 periods. *§:* `platform.md` §7; `operations.md` §14.

- [ ] **A/B rollback proven in production (canary → cohort → fleet) with ZERO hot-water incidents.**
  *Verified by:* **⛔ Steps 25, 26, 27, 28** (HIL bench → blocking safety suite → production rollback through staged rollout → zero-hot-water release gate), backed by Steps 4 (node A/B + regression watchdog), 5 (thermal guard), 11 (HeaterIface fail-safe), 18 (staged OTA + auto-halt). *Metric:* S1 = 0 incidents; S14 OTA-attributable thermal faults = 0; rollback success ≥99.5%. *§:* `operations.md` §12–§13; `node.md` §12, §16; R9.

- [ ] **Marketplace-concentration metric instrumented and reported (<60%).**
  *Verified by:* Step 21 (per-marketplace revenue share instrumentation) reading from Step 23 (ledger/tape). *Metric:* trailing-30-day max single-platform revenue share reported; <60% covenant trigger. *§:* `platform.md` §16; `financial-and-roadmap.md` §C2.4 (R6).

**Also de-risked in Phase A (measurement deliverables for underwriting — `financial-and-roadmap.md` §D2):** R2 (relay-bytes vs direct-bytes + relay-$/GPU-hr, Steps 10/12), R5 (per-region seasonal thermal-overlap from telemetry, Steps 8/14), R8 (tape integrity foundation, Steps 22–24/29), R9 (recovery-ladder chaos + zero-hot-water gate, Steps 26–29), R11 (UL/ETL listing + reference permit in the pilot jurisdiction), R12 (real homeowner churn/removal rate from registry lifecycle).

---

*Cross-references:* `node.md` (OS/agent/A-B/thermal guard/recovery ladder/attestation), `connectivity-and-data.md` (overlay + relay + cache), `platform.md` (registry, telemetry, OTA, vast.ai connector, metering ledger/tape, concentration guard), `data-contracts.md` (NORMATIVE telemetry/usage/job/score/thermal schemas), `operations.md` (HIL rig, OTA gating, zero-hot-water release gate, observability, field ops), `financial-and-roadmap.md` (Phase A scope §C2, exit criteria §C2.4, risk register §D).

---

## Part 3 — Phase B: Own the orchestration

**Status:** Build plan · **Phase:** B (of A→B→C) · **Audience:** Engineering, SRE, finance-systems
**One-line outcome:** Capture the marketplace fee, build a real utilization floor, and turn the tape from "reported by marketplaces" into an **authoritative, owned, double-signed metering ledger** — the securitization tape proper. Unlocks the **warehouse facility** (funding ladder rung 2). (`financial-and-roadmap.md` §C3.)

**Carried-in from Phase A (do not rebuild):** node agent + immutable A/B OS (`node.md`), outbound-only WireGuard overlay + DERP PoPs (`connectivity-and-data.md`), telemetry pipeline + minimal registry (`platform.md` §1–§2), vast.ai connector, metering v0 (read-only tape *as reported by the marketplace*, `financial-and-roadmap.md` §C2). Phase A produced 12 months of clean per-unit tape on ≥1,000 units — the pre-engagement artifact for the warehouse lender.

**Authority rule for this part:** where a step touches a shared object, the cited `data-contracts.md` § is **NORMATIVE** and wins. `platform.md` is authoritative for *mechanism*; `operations.md` §1 (S1–S20) is authoritative for the *SLO numbers* we build and test against.

**How to read a step:** each has **Action** (concrete, tech), **Prereq**, **Done when** (acceptance), **Doc §**. Steps flagged **🔬 SIM-GATED** must be validated in the fleet-scale simulator/replay/HIL *before* production per `operations.md` Part B. Steps flagged **💰 MONEY** are on the exact (zero-tolerance) budget, never the lossy telemetry budget (`operations.md` §0 two-budget rule).

---

### Workstream 0 — Foundations (do first; everything else depends on these)

- [ ] **B0.1 — Make `fits()` the single shared implementation.**
  **Action:** Extract the canonical `fits(job, node, window_remaining_s)` predicate into one library (tier-order, confidentiality, gpu_count, non-interruptible whole-window fit, interruptible `max(min_thermal_window_s, T_GRACE_S=30)`); import it identically in scheduler, yield router, utilization engine. Add the `tier_order()` comparison and reliability-score formula as pure functions next to it.
  **Prereq:** Phase A registry; `data-contracts.md` adopted.
  **Done when:** one golden-vector corpus produces *identical* verdicts across all three callers (conformance test `operations.md` §8.3); unit tests cover `expected_duration_s=None`→false, gpu_count>free→false, trusted_host mismatch→false. **🔬 SIM-GATED**
  **Doc §:** `data-contracts.md` §7, §2, §3; `operations.md` §8.1, §8.3.

- [ ] **B0.2 — Pin tunables in registry config.**
  **Action:** Add config rows for `TARGET_UPLINK_MBPS`, `PREEMPT_CAP`, `FAULT_CAP`, per-SKU `usable_thermal_kwh` (H1 = 8.8 kWh; never a hardcoded `12.0`), `T_GRACE_S=30`, reconciliation `tolerance` (target ±2%), concentration `warn_share=0.50`/`max_share=0.60`/`warn_penalty`/`over_cap_factor`, backlog reservation price `$0.05/GPU-hr`.
  **Prereq:** B0.1.
  **Done when:** conformance test fails the build if any `12.0` literal appears (`operations.md` §8.3); thermal coordinator imports `usable_thermal_kwh` per SKU.
  **Doc §:** `data-contracts.md` §6, §9; `platform.md` §6.1.

- [ ] **B0.3 — Stand up the fleet-scale simulator (DES) as a first-class CI tier.**
  **Action:** Build the discrete-event simulator that runs the *real* scheduler/thermal-coordinator/yield-router binaries against a synthetic fleet (thermal twin per node implementing `T_tank(t+Δt)=T_tank+(P_in−D−loss)·Δt/C_th`, Δt=5 min). Implement the scenario file I/O spec and the measured-output report (placement p95, idle-leak, wrongful preempt, util%, overlap%, comfort violations, window-pred error, oracle gap, concentration share).
  **Prereq:** B0.1.
  **Done when:** the 7 required scenarios (steady commercial winter, residential summer DHW-only, mixed shoulder, spot drought, concentration pressure, preemption storm, scale ramp 1k→100k) run nightly, seeded/reproducible, emit machine-readable reports diffable by CI. **🔬 SIM-GATED**
  **Doc §:** `operations.md` §9 (architecture, I/O spec, scenario matrix).

---

### Workstream 1 — Scheduler / Orchestrator (the operational core)

- [ ] **B1.1 — Persist the normalized job + assignment model.**
  **Action:** Create `job` and `job_assignment` tables as the projection of `superheat.job.v1`; implement the lifecycle FSM `QUEUED→MATCHED→DISPATCHED→STARTING→RUNNING→{COMPLETED|PREEMPTED|FAILED}` with `end_reason ∈ {completed,preempted_thermal,preempted_yield,fault}`. Every transition emits a usage record (wire to B3).
  **Prereq:** B0.1; Phase A registry.
  **Done when:** integration test drives all transitions; PREEMPTED path requeues; conformance round-trips `superheat.job.v1` (full `source` enum, seconds, `RTX4090` token).
  **Doc §:** `platform.md` §5.1, §5.3; `data-contracts.md` §4.

- [ ] **B1.2 — Implement `place(job)` matching: hard filter then score.**
  **Action:** Candidate set from registry (`active`, `attested`, gpu_model, free_gpus); **hard filter** with `fits()` against `thermal_coordinator.window(n).usable_seconds`; **score** survivors with `α·reliability + β·locality + γ·thermal_fit + δ·net_quality − ε·preempt_risk − ζ·fragmentation` (start α=1.0,β=25,γ=15,δ=10,ε=40,ζ=20); `argmax`, then **reserve gpu rows in a Postgres txn** and dispatch over the outbound stream. Priority queue: premium/reserved first, backlog lowest.
  **Prereq:** B1.1, B0.1, thermal coordinator window RPC (B2.3).
  **Done when:** integration test asserts no double-booking under concurrent `place()` (the `gpu_one_live_per_slot` unique index + `gpu.state` txn); sim shows placement **p95 < 2 s** for available capacity (S5). **🔬 SIM-GATED**
  **Doc §:** `platform.md` §5.2; `operations.md` §8.2, §9.2 (S5).

- [ ] **B1.3 — Anti-thrash / hysteresis.**
  **Action:** A feasible incumbent is displaced only if a challenger beats it by margin (start >10%) sustained over a debounce interval (≥60 s); EWMA-smooth tier-cache updates before they can demote a running node; never re-place a job that wasn't actually preempted.
  **Prereq:** B1.2; reliability tier cache (B4.4).
  **Done when:** preemption-storm sim shows no placement churn / no wrongful non-interruptible eviction (S7 <0.5%). **🔬 SIM-GATED**
  **Doc §:** `platform.md` §5.2; `platform.md` §29 (same hysteresis on yield preemption).

- [ ] **B1.4 — Edge-cached reads, central transactional reservations (scale).**
  **Action:** Let regional edge aggregators cache READ-ONLY thermal-window snapshots + reliability scores for low-latency scoring near nodes; keep the `gpu` row reservation a single central Postgres txn (a stale edge cache may cause a rejected placement attempt, never a double-booking). Stateless scheduler workers behind a queue, sharded by `node.region`.
  **Prereq:** B1.2.
  **Done when:** scale-ramp sim (1k→10k→100k) keeps **p95 < 2 s** (S5); thundering-herd chaos (100k simultaneous arrivals) degrades to backpressure, never double-books. **🔬 SIM-GATED**
  **Doc §:** `platform.md` §9; `operations.md` §16 (thundering herd).

- [ ] **B1.5 — Idle-window-leak guard.**
  **Action:** Ensure any usable thermal window with no paying job and a non-empty backlog is filled within 5 min (wire to backlog filler, B6).
  **Prereq:** B1.2, B6.
  **Done when:** spot-drought sim shows **idle-window leak < 1%** of window-minutes (S8). **🔬 SIM-GATED**
  **Doc §:** `platform.md` §5.4; `operations.md` §9.2 (S8).

---

### Workstream 2 — Thermal Coordinator (tank-as-battery, pre-heat)

- [ ] **B2.1 — Per-site tank-as-battery state model.**
  **Action:** Maintain per-node state at Δt=5 min: `T_tank, T_min, T_max, C_th, E_store=C_th·(T_max−T_tank), E_avail, SoC`. Import `usable_thermal_kwh` per SKU (B0.2). Compute `storage_headroom_kwh = E_store·(T_max−T_current_avg)/ΔT_usable`, clamped ≥0.
  **Prereq:** B0.2; warm-tier telemetry (Phase A).
  **Done when:** unit tests reproduce `C_th≈0.352 kWh/°C`, `E_store≈8.8 kWh` for 80-gal H1; headroom clamps ≥0; no `12.0` literal. **🔬 SIM-GATED**
  **Doc §:** `data-contracts.md` §6; `platform.md` §6.1, §20.1–§20.2; `operations.md` §8.1.

- [ ] **B2.2 — Heat-demand forecast + pre-heat scheduler (the control law).**
  **Action:** Build the per-site forecaster `D(t)` over a 6-hour rolling horizon (72 intervals) from learned occupancy + weather/HDD + season, keyed by node `timezone`. Implement `preheat_start(peak_t, peak_kWh)` with the **feasibility guard** `bankable_kWh = min(peak_kWh, E_store)` + standby-loss makeup + `PARTIAL_PREHEAT` flag when peak > storage. Emit per node/interval a **runnable-minutes budget** + **mode hint** (`discharge|charge|preheat|hold`).
  **Prereq:** B2.1.
  **Done when:** unit tests: `preheat_start` never schedules banking > `E_store`; replay test (Phase-A telemetry) shows **window-prediction error p50 within ±15%** (S18); the control law is validated in sim before production. **🔬 SIM-GATED**
  **Doc §:** `platform.md` §6.2, §20.3–§20.4, §21; `operations.md` §10.1, §9.2 (S18).

- [ ] **B2.3 — Scheduler-facing interface (hard filter, never bypassed).**
  **Action:** Implement `ThermalCoordinator` gRPC: `GetWindow(NodeId)→ThermalWindow{can_run_now, usable_seconds, soc, preheat_requested, max_continuous_w}` and `StreamWindows(RegionId)`. Scheduler treats the window as a **hard filter** and never computes thermal feasibility itself. Mark window CLOSED if honoring a job risks comfort or exceeds `max_safe_c` (the node agent enforces locally regardless — principle 2).
  **Prereq:** B2.2.
  **Done when:** integration test: scheduler `place()` hard-filters on a live `ThermalWindow`; sim asserts **comfort violations attributable to compute = 0** (S1, hard). **🔬 SIM-GATED**
  **Doc §:** `platform.md` §6.3–§6.4; `operations.md` §9.2 (S1, S18).

- [ ] **B2.4 — Measure `η_capture` / `C_th` on the HIL rig (close the placeholder).**
  **Action:** On the per-region HIL bench with a real tank, measure `η_capture` (placeholder 0.95) and `C_th`; feed measured values back into B2.1 and the sim. Run the sim's `η_capture ∈ [0.85, 0.97]` sensitivity sweep so finance sees a band, not a point.
  **Prereq:** HIL rig (carried from Phase A, R9 invariant); B0.3.
  **Done when:** measured constants replace placeholders in registry config; sensitivity band documented for underwriting; replay-vs-sim overlap gap is not material. **🔬 SIM-GATED**
  **Doc §:** `data-contracts.md` §6; `operations.md` §12.2, §10.2.

---

### Workstream 3 — Authoritative Ledger / Securitization Tape  💰 MONEY

- [ ] **B3.1 — Double-signed usage records become the billing primitive.**
  **Action:** Node agent emits `superheat.usage.v1` at job stop / checkpoint / heartbeat-roll: **per-GPU** (`gpu_slot` mandatory), with `gpu_seconds`, `energy_kwh`, `gross_usd/fee_usd/net_usd`, `node_sig`. Control plane verifies and **counter-signs** (`cloud_sig`) on ingest (after reconciliation, B5). Records are append-only and **never span a billing-period boundary** (split at boundary). This replaces metering v0 — billing is no longer derived from telemetry.
  **Prereq:** Phase A signing keys (`security-and-compliance.md`); B1.1.
  **Done when:** property tests: unsigned/single-signed/replayed (dup `usage_id`)/tampered records rejected at counter-sign; `gross == fee + net`; no record spans a month boundary; a boundary-crossing job yields exactly two adjacent records summing to the original. **🔬 SIM-GATED**
  **Doc §:** `data-contracts.md` §8; `platform.md` §7.1; `operations.md` §14.1 (S10).

- [ ] **B3.2 — Append-only ledger tables.**
  **Action:** Create `usage_record` (FK `gpu_id` to the exact physical card + `gpu_slot`, indexed `(node_id,gpu_slot,start_ts)` and `(tranche_id,start_ts)`), `settlement`, and `tranche`. Every node/GPU-hour carries its `tranche_id` from day one (warehouse/ABS segregation).
  **Prereq:** B3.1; registry `gpu`/`tranche`.
  **Done when:** 100% of paid GPU-hours backed by a double-signed record (S10, zero budget); every row carries a financing tag.
  **Doc §:** `platform.md` §7.2; `financial-and-roadmap.md` §C1 principle 5.

- [ ] **B3.3 — The corrected per-GPU securitization tape view.**
  **Action:** Build `securitization_tape` materialized view: aggregate **per GPU first** (`gpu_period` CTE joining `usage_record` to `gpu` on **both** `node_id` AND `gpu_slot` — the Cartesian-multiply fix), then roll up to node rows. Emit origination_date, tranche, GPU-hours, gross, realized $/GPU-hr, reliability track record, and true `utilization_pct` (earning GPU-seconds ÷ installed GPUs × seconds in period). Filter `lifecycle_state='active' AND seasoned=true`.
  **Prereq:** B3.2; reliability scoring (B4).
  **Done when:** golden tests pin exact tape rows incl. month-boundary job, multi-GPU node (total == Σ per-GPU, no multiply), module-swap mid-period, and unseasoned node correctly **excluded**; idempotent rebuild from append-only records reproduces an identical tape (S12). **🔬 SIM-GATED**
  **Doc §:** `platform.md` §7.3; `data-contracts.md` §8; `operations.md` §14.1–§14.2.

- [ ] **B3.4 — Extend tape + settlement with OPEX and refresh-reserve (DSCR must be measurable).**
  **Action:** Add `opex_allocated_usd` and `refresh_reserve_usd` to `settlement` and `securitization_tape` so DSCR is measurable **on net-of-OPEX** (the covenant is measured there) and the refresh reserve is provably funded.
  **Prereq:** B3.3; OPEX model (`financial-and-roadmap.md` §B2–§B6).
  **Done when:** tape reports net-of-OPEX per node/period; finance can compute trailing-6-mo DSCR directly off the tape (feeds exit criterion).
  **Doc §:** `financial-and-roadmap.md` §B6, §C3; `platform.md` §7.

---

### Workstream 4 — Reliability Scoring + Tiering

- [ ] **B4.1 — Scoring service as a telemetry consumer + daily batch.**
  **Action:** Compute windowed inputs (`uptime_frac` excl. `cloud_degraded` windows, `thermal_avail`, `net_quality`, `preempt_penalty`, `fault_penalty`) and the canonical score `base=0.45·uptime+0.25·thermal_avail+0.20·net_quality+0.10·(1−preempt_penalty)`, `score=round(100·base·(1−0.30·fault_penalty))`, then EWMA-smooth (α=0.3 daily) and clamp [0,100].
  **Prereq:** Phase A telemetry; `cloud_degraded` stamping (carried in).
  **Done when:** property test — perfect node == **exactly 100**; fault caps at −30%; `cloud_degraded` excluded from uptime; replay test asserts no tier whipsaw on a single bad day. **🔬 SIM-GATED**
  **Doc §:** `data-contracts.md` §3; `platform.md` §4.1–§4.2; `operations.md` §8.1, §10.1.

- [ ] **B4.2 — Tier ladder + seasoning gate.**
  **Action:** Derive tier from score (≥85 premium / ≥70 standard / ≥50 spot / else probation). Force `probation` + `seasoned=false` for any node with <30 days clean history (not tape-eligible). Persist `node_reliability` + daily `node_reliability_history` (feeds the tape track record).
  **Prereq:** B4.1.
  **Done when:** unit test — seasoning gate forces probation regardless of score; tape join (B3.3) sees only `seasoned=true`.
  **Doc §:** `data-contracts.md` §2, §3; `platform.md` §4.2–§4.3, §24.1.

- [ ] **B4.3 — Wire tiering into routing gates.**
  **Action:** Scheduler + yield router gate work class via the single `tier_order` comparison inside `fits()`: non-interruptible/contracted → premium/standard; spot/interruptible/backlog → spot/probation.
  **Prereq:** B4.2, B0.1.
  **Done when:** conformance — all three engines use the same `fits()`; `trusted_host` never appears in `reliability_tier_required`.
  **Doc §:** `data-contracts.md` §2, §7; `platform.md` §24.2.

- [ ] **B4.4 — Score freshness + tier propagation SLO.**
  **Action:** Recompute ≥ daily/node; push tier changes to the scheduler cache (EWMA-smoothed before demoting a running node).
  **Prereq:** B4.1, B1.3.
  **Done when:** integration test — tier change propagates to scheduler cache **< 5 min** (S9).
  **Doc §:** `platform.md` §4.4; `operations.md` §9.2 (S9).

---

### Workstream 5 — Yield Router v1 (with the 60% concentration guard)

- [ ] **B5.1 — Connector abstraction + RunPod & Salad connectors.**
  **Action:** Implement the `MarketplaceConnector` protocol (`quote/list/unlist/poll/normalize/on_assigned/preempt/heartbeat/pull_settlement`) and `SettlementProfile` (fee, payout latency, currency, fx_volatility, dispute window). Add RunPod (Community) + Salad connectors; vast.ai carries over from Phase A. Each `normalize()` output must be fed unmodified to the scheduler.
  **Prereq:** B1.1; `superheat.job.v1`.
  **Done when:** conformance — each connector round-trips `superheat.job.v1` (full enum, seconds, exact `RTX4090`, `net = gross·(1−fee)`); integration — normalized job runs through scheduler unmodified.
  **Doc §:** `platform.md` §12–§13; `data-contracts.md` §4; `operations.md` §8.2–§8.3.

- [ ] **B5.2 — Expected-value engine.**
  **Action:** Compute `E[$/gpu-hr](s,node) = net_price · p_fill · expected_served_fraction(W) · latency_factor − expected_preempt_cost`, normalized per-GPU-hour so windows of different lengths compare. `expected_preempt_cost` uses bandwidth-aware checkpoint/restage estimates from `connectivity-and-data.md`.
  **Prereq:** B5.1; thermal window (B2.3); checkpoint cost model (B7).
  **Done when:** unit test reproduces the worked example table (vast≈0.226, runpod≈0.167, salad≈0.143, direct≈0.155, backlog 0.050); served-fraction clamps when window < runtime. **🔬 SIM-GATED**
  **Doc §:** `platform.md` §16.1–§16.2, §16.5; `operations.md` §8.1.

- [ ] **B5.3 — Routing decision loop.**
  **Action:** Implement `route_node_hour`: (0) honor reserved-floor via `fits()` first; (1) gather cached/parallel quotes; (2) hard-filter with `fits()`; (3) apply concentration guard; (4) always append always-commitable backlog; (5) pick max EV; (6) commit on chosen source, retry next-best on listing-race loss.
  **Prereq:** B5.2, B6, B4.3.
  **Done when:** integration — losing a listing race falls through to next-best; backlog always wins when all external fail.
  **Doc §:** `platform.md` §16.3.

- [ ] **B5.4 — Concentration guard: soft penalty + HARD 60% covenant stop.**  💰 MONEY
  **Action:** Two layers: (a) graduated **soft** penalty between `warn_share=0.50` and `max_share=0.60`; (b) **hard** stop — no new placement to a marketplace at/over 60% of trailing-30d revenue, *unless* the only alternative is idling, in which case place + `flag_covenant_exception` for the monitor. `direct/reserved/backlog` are exempt.
  **Prereq:** B5.3; ledger revenue-share query (B3).
  **Done when:** unit test — at 62% share vast is dropped (hard stop) and RunPod wins; graduated penalty between 0.50–0.60; exempt classes pass. Concentration-pressure sim shows trailing-30d max single-marketplace share **≤ 60%** without idling nodes (the **60%×70% claim** family — validate in sim before production). **🔬 SIM-GATED**
  **Doc §:** `platform.md` §16.3 (`apply_concentration_guard`), §16.5; `operations.md` §9.2–§9.3 (scenario 5).

- [ ] **B5.5 — Reserved-floor planner (begin) + reserved demand intake.**
  **Action:** Nightly planner pre-allocates high-reliability node-hours across the month to meet reserved monthly commitments (spread, not front-loaded), releasing unused allocation back to spot. Reserved jobs are non-interruptible, `reliability_tier_required ≥ standard`. (Full reserved-contract productization is Phase C; B begins the planner to lift contracted-revenue ratio.)
  **Prereq:** B5.3; B4.
  **Done when:** sim — pre-allocated reserved windows are honored at step 0; contracted GPU-hours count toward the ≥30% exit criterion. **🔬 SIM-GATED**
  **Doc §:** `platform.md` §14, §17; `financial-and-roadmap.md` §C3.

---

### Workstream 6 — Interruptible Batch/Inference Backlog Filler

- [ ] **B6.1 — Backlog source + always-available filler.**
  **Action:** Stand up the `source="backlog"` / `tenant_class="internal_backlog"` connector for Superheat-owned/partner batch inference, fine-tuning, embedding/eval work. Properties: `interruptible=true`, `preempt_penalty=0`, `checkpointable=true`, `min_thermal_window` very small (5 min) so it fits any window; `net_price = $0.05/GPU-hr` reservation price (electricity floor) so it never displaces real revenue, only prevents idle.
  **Prereq:** B5.1 (same connector interface); B0.2 (`$0.05`).
  **Done when:** router falls back to backlog for any node/window; spot-drought sim shows the floor held + idle-leak < 1% (S8). **🔬 SIM-GATED**
  **Doc §:** `platform.md` §15, §25 (L1); `operations.md` §9.3 (scenario 4).

- [ ] **B6.2 — Backlog depth management.**
  **Action:** Size/refill the queue so depth always exceeds fleet idle capacity; if it empties, fall through to DR/VPP (B8) as absolute floor (L4).
  **Prereq:** B6.1; B8.
  **Done when:** monitoring shows queue depth > idle capacity; empty-backlog path routes to DR.
  **Doc §:** `platform.md` §15, §25 (L4).

---

### Workstream 7 — Checkpoint / Resume v1

- [ ] **B7.1 — Three-stage checkpoint decoupled from preemption.**
  **Action:** Implement Stage 1 local NVMe flush, Stage 2 **async upload to nearest PoP** (content-defined ~4 MB rolling-hash chunking + dedup vs previous checkpoint), Stage 3 register-durable ("job J has durable checkpoint at step n @ pop:<region>, digest:<…>"). Reserve a fixed local slice (~150 GB) for checkpoint scratch so a checkpoint can always be written even when cache is full.
  **Prereq:** Phase A overlay + PoP cache; node agent.
  **Done when:** the expensive upload happens during the job, not at preemption; scheduler can preempt only after Stage 3 for the latest step.
  **Doc §:** `connectivity-and-data.md` §B2.1–§B2.2, §B1.

- [ ] **B7.2 — Graceful-stop protocol with `T_GRACE_S=30`.**
  **Action:** On preempt, `signal(PREEMPT)` → `wait_for_checkpoint(T_grace=30 s)` to flush in-flight to NVMe → mark RESUMABLE if checkpointed; else for COMFORT_SAFETY hard-kill + REQUEUE (idempotent), else bounded extension. Durable resume path is always the last **PoP-persisted** checkpoint (30 s can't finish a multi-minute upload).
  **Prereq:** B7.1; preemption triggers in scheduler/coordinator.
  **Done when:** preemption-storm chaos — graceful-stop honored, resume from durable PoP checkpoint, no wrongful non-interruptible eviction (S7). **🔬 SIM-GATED**
  **Doc §:** `data-contracts.md` §7 (T_GRACE_S=30); `platform.md` §26; `operations.md` §16.

- [ ] **B7.3 — Location-independent resume (migration, not loss).**
  **Action:** Resume from the content-addressed checkpoint on *any* node with a window; new node **downloads** from PoP over its fast downlink; bandwidth-aware placement biases data-heavy jobs to better-connected nodes.
  **Prereq:** B7.1; scheduler placement (B1.2).
  **Done when:** preempted job resumes useful compute within **5 min** (p50 ≤2 min, p95 ≤10 min) for working sets ≤40 GB, RPO ≤ one checkpoint interval (default ≤15 min); checkpoint-success-rate / mean preemption-recovery-time reported (Phase-B KPI). **🔬 SIM-GATED**
  **Doc §:** `connectivity-and-data.md` §B2 (RTO/RPO targets); `platform.md` §26; `financial-and-roadmap.md` §C3 KPIs.

---

### Workstream 8 — DR / VPP Enrollment (begin)

- [ ] **B8.1 — DR/VPP connector + enrollment (start).**
  **Action:** Begin enrolling node flexible load with EnergyHub (already aggregates Rheem water heaters across 30+ utilities) / Renew Home. Model DR as L4: a *charge* event = free pre-heat opportunity; a *shed* event = GPU-off (already the comfort-safe default). DR only claims hours the upper layers declined (DR and compute never fight).
  **Prereq:** Thermal coordinator (B2); backlog floor (B6).
  **Done when:** first nodes enrolled; `$/unit-yr enrolled` reported as a Phase-B KPI; DR shed/charge events flow through the control loop without contending with compute. (Fleet-scale VPP is Phase C.)
  **Doc §:** `platform.md` §25 (L4), §28.2; `financial-and-roadmap.md` §C3 (DR/VPP begin), §B5.1 (summer floor).

---

### Workstream 9 — Reconciliation + Release Gates (Phase-B financial gate)  💰 MONEY

- [ ] **B9.1 — Automated settlement reconciliation pipeline.**
  **Action:** `pull_settlement()` per source → match `PayoutRecord.source_job_ref` to usage records → buckets: RECONCILED (write `cloud_sig`, post owner-settlement), FLAG variance (>tolerance), ORPHAN (payout, no record), MISSING (record, no payout past SLA → arrears). Currency-normalize token payouts (io.net/Akash) at realized FX; fee truth-up updates `SettlementProfile`; hold settlement on breach.
  **Prereq:** B3, B5.1.
  **Done when:** the ±tolerance applies to **signed usage records vs marketplace settlement** (never telemetry); drift > tolerance opens an audit ticket + **blocks settlement** within one period (S11); orphan/missing/variance produced exactly for injected anomalies, zero false positives on clean data. **🔬 SIM-GATED**
  **Doc §:** `platform.md` §18; `data-contracts.md` §8; `operations.md` §14.3.

- [ ] **B9.2 — Three-way reconciliation harness (the R8 Phase-B exit gate).**
  **Action:** Build the harness reconciling **usage records ↔ marketplace settlement ↔ bank cash receipts** to ±2% tolerance, producing the external-audit-ready variance report. Assert telemetry is never the arbiter; per-GPU join correctness; dispute/clawback window (RECONCILED ≠ FINAL until `dispute_window_days`).
  **Prereq:** B9.1; B3.4 (bank receipts feed).
  **Done when:** three-way drift ≤ **±2%**, opened < 1 reporting period after close (S11); monthly close < 24 h, idempotent rebuild (S12); harness is wired as a CI gate. **🔬 SIM-GATED**
  **Doc §:** `operations.md` §14.3, §17; `platform.md` §7.4–§7.5; `financial-and-roadmap.md` §C3.

- [ ] **B9.3 — Wire the moat services into the zero-hot-water release gate.**
  **Action:** No Phase-B service (scheduler, coordinator, router, ledger) promotes past sim-nightly/HIL/staging unless: unit math green (score==100, `fits()`, period-split, no `12.0`), conformance green, sim hard SLOs green (comfort=0, p95<2s, idle-leak<1%, concentration≤60%), HIL safety suite green (BLOCKING), chaos green, tape correctness + three-way reconciliation green.
  **Prereq:** B0.3, B9.2, HIL rig.
  **Done when:** the `operations.md` §13.2 checklist passes for every Phase-B release; canary excludes non-interruptible reserved nodes. **🔬 SIM-GATED**
  **Doc §:** `operations.md` §13, §11–§12; (S1, S5, S7, S8, S10–S12).

---

### Phase B Exit Checklist — criterion → verifying step

Exit criteria from `financial-and-roadmap.md` §C3.4 (= warehouse-facility entry). Each maps to the step(s) that verify it.

- [ ] **1. Owned ledger reconciles within tolerance** (usage records ↔ marketplace settlement ↔ bank receipts, ±2%).
  → **B3.1 + B3.3 + B9.1 + B9.2** (double-signed primitive → corrected tape → automated + three-way reconciliation harness, S10–S12). 💰
- [ ] **2. Trailing-6-month DSCR ≥ 2.5×** on the candidate pool (measured net-of-OPEX).
  → **B3.4** (tape carries `opex_allocated_usd`/`refresh_reserve_usd` so DSCR is measurable net-of-OPEX) + **B3.3** (correct per-GPU GPU-hours/revenue). DSCR is a finance computation off the corrected tape. 💰
- [ ] **3. ≥30% of GPU-hours reserved/contracted** (reserved-instance + HSA + DR).
  → **B5.5** (reserved-floor planner) + **B8.1** (DR/VPP enrollment begun); contracted share measured via tape revenue-by-source (B3.3).
- [ ] **4. Warehouse ≥70% utilized against the seasoned fleet; rating-agency pre-read complete.**
  → **B4.2** (seasoning gate → seasoned sub-pool) + **B3.3** (true `utilization_pct`) + **B2.2/B6.1/B8.1** (pre-heat + filler + DR lift the utilization floor); the **60%×70% claim** is validated in **B0.3 + B2.2 + B5.4 + `operations.md` §10.2** sim/replay before it is quoted. 🔬
- [ ] **5. Backup-servicer runbook / code escrow drafted (R7).**
  → `operations.md` Part A §6 is the human-readable escrow deliverable; the escrow package (this build plan + design docs + key custody + first-72-hours order + failover-drill record) is assembled in Phase B; the servicer-failover drill itself is run before the inaugural ABS (Phase C). Verified against **B9.2** (reconciliation discipline a replacement servicer must not break) + DR drill (`operations.md` §5).

**Items validated by simulation before production** (per `operations.md` Part B): scheduler SLO (B1.2/B1.4, p95<2s, S5/S7/S8), thermal control law (B2.2/B2.3/B2.4, window-pred ±15% + comfort=0, S18/S1), and the **60%×70% utilization×overlap claim and 60% concentration guard** (B0.3/B2.2/B5.4 + `operations.md` §9.3 scenarios 1–5, §10.2). None of these numbers may be promised to a financier until they are earned in the simulator and confirmed in replay.

---

*Cross-references:* `platform.md` (scheduler §5, thermal coordinator §6, reliability §4, ledger/tape §7, yield router §16, concentration guard §16.3, backlog §15, utilization layers §25, reserved planner §17), `data-contracts.md` (NORMATIVE: job §4, telemetry §5, usage §8, reliability §2–§3, thermal §6, `fits()` §7), `connectivity-and-data.md` (checkpoint/resume §B1–§B2), `operations.md` (SLO catalog §1, simulator §9, control-system validation §10, HIL §12, release gate §13, tape/reconciliation §14, chaos §16, backup-servicer §6), `financial-and-roadmap.md` (Phase B scope/exit §C3, OPEX→tape linkage §B6, DSCR §B4).

---

## Part 4 — Phase C: Own the demand & UX

**Status:** Build plan · **Audience:** Eng leads, squad ICs, finance-systems, field-ops, security · **Phase window:** ~24 → 36 mo (`financial-and-roadmap.md` §C4, C5)

**One-line goal.** Own the demand channel and the renter experience: maximize realized $/GPU-hr and the contracted-revenue ratio so the fleet becomes a *ratable, diversified collateral package* for the inaugural term ABS (rung 3–4) and programmatic issuance (rung 5). (`financial-and-roadmap.md` §C4.)

**Phase C exit criteria (from `financial-and-roadmap.md` §C4.4) — every step below maps to one of these:**
1. ≥10,000 seasoned units (≥12-mo metered history) available as collateral.
2. Inaugural-ABS readiness: rising contracted-revenue share over the deal's life.
3. Executed + **tested** backup-servicer / servicer-replacement (R7).
4. Cache-driven relay-cost reduction demonstrated (controls the R2 OPEX line).

**Invariants that do not change in Phase C** (`financial-and-roadmap.md` §C1.3): outbound-only; fail-safe-to-heater; thermally-coupled compute; measure-then-price; hybrid demand. No Phase-C feature may open an inbound port, brick a Phase-A/B unit, or defeat the thermal interlock. `data-contracts.md` is NORMATIVE — every schema below is a projection, never a fork.

**Entry preconditions (must be true before Phase C starts):** Phase B exit met — authoritative owned ledger reconciles within tolerance; trailing-6-mo DSCR ≥ 2.5×; ≥30% fleet GPU-hours contracted; warehouse ≥70% utilized and drawn; backup-servicer runbook *drafted* (R7); checkpoint/resume v1, yield-router v1, reliability tiering all live. (`financial-and-roadmap.md` §C3.4, §C4.3.)

The work is organized into **8 workstreams**. Steps are globally numbered C-1 … C-31 and ordered so prerequisites precede dependents. Run workstreams in parallel across squads; the exit checklist at the end maps each criterion to its verifying step.

---

### Workstream 1 — Renter API + SDK (the direct-demand surface)

> Owner: Product + backend + frontend. Implements `platform.md` Part D (§31–§37). This is the revenue surface that captures the 25–35% marketplace fee back as margin (`platform.md` §14).

#### C-1. Stand up the direct renter REST/gRPC API
- **Action.** Build `api.superheat.io` (REST) + `grpc.superheat.io:443` (`superheat.renter.v1`). Implement the job endpoints `POST /v1/jobs`, `GET /v1/jobs/{id}`, `GET /v1/jobs`, `POST /v1/jobs/{id}:cancel`, `…/logs`, `…/exec`, `…/events` (`platform.md` §32.2). `RenterJobSpec` (§31.1) is validated then projected to canonical `superheat.job.v1`; server injects `source:"direct"`, `tenant_class:"direct"`, `economics.platform_fee_frac=0`, `trust.egress_policy:"restricted"`, `trust.no_persistence:true` (`platform.md` §31.2). Reject `reliability_tier:"probation"` with `422 tier_not_rentable`.
- **Prereqs.** Phase B yield router + scheduler (already consume `superheat.job.v1`); `data-contracts.md` §4 frozen.
- **Done when.** A `POST /v1/jobs` with a valid spec produces a canonical job that reaches the *unchanged* yield router (§16) and scheduler (§5) with `source="direct"`; contract-conformance test (`operations.md` §8.3) round-trips the projection with zero field loss.
- **Design-doc §.** `platform.md` §31–§32; `data-contracts.md` §4, §9.

#### C-2. Auth, secrets, idempotency, rate limits, error model
- **Action.** Implement API-key (`sk_live_`/`sk_test_`, scoped `jobs:*`/`volumes:*`/`billing:read`, shown once) + OAuth2 client-credentials (`POST /v1/oauth/token`), TLS 1.3 mandatory, mTLS optional (`platform.md` §32.1). `POST /v1/secrets` (KMS-backed; referenced as `{"secret":"name"}` in `env` so plaintext never transits — §32.8). `Idempotency-Key` stored 24 h, verbatim replay on same key+body, `409 conflict` on key+different body (§37.4). Rate limits/quotas (§37.1) + RFC-9457 problem+json error model with stable `code`s (§37.3).
- **Prereqs.** C-1.
- **Done when.** Retried `superheat run` never double-submits (idempotency test); `429`/`409 quota_exceeded`/`no_capacity` return the documented codes; secrets never appear in logs or job spec.
- **Design-doc §.** `platform.md` §32.1, §32.8, §37.

#### C-3. Price-quote + billing endpoints, honest confidentiality block
- **Action.** `POST /v1/quotes` returns indicative `net_usd_per_gpu_hr` with `breakdown` (base + tier_premium + confidentiality_premium) and short `valid_until` TTL (`platform.md` §32.6); `GET /v1/usage` (projection of `superheat.usage.v1`), `GET /v1/billing/invoices`. Every Job view echoes `reliability_tier` and a `confidentiality_guarantee` block stating `hardware_tee:false` + the `guarantees`/`not_guaranteed` lists on standard nodes (§32.4, §33.3) — never silently downgrade a `trusted_host` request to `standard` (`409 no_capacity` instead).
- **Prereqs.** C-1; reliability tiering (Phase B); trusted-host registry flag (C-22).
- **Done when.** A `trusted_host` quote with no eligible node returns `409`/`confidentiality_unsatisfiable`; the standard-node Job view always carries the no-TEE honesty block.
- **Design-doc §.** `platform.md` §32.4, §32.6, §33.3; `security-and-compliance.md` §4.1, §4.3.

#### C-4. SDK + `superheat` CLI + events/webhooks
- **Action.** Ship Python SDK (`sh.Client`, `client.jobs.run(...)`, `job.events()`, `job.results.download()`) and the `superheat` CLI verbs `run/ssh/ls/logs/cp/ckpt/cancel/quote/volume` (`platform.md` §33.1–§33.2). Ship `superheat-ckpt` (PyTorch/HF/DeepSpeed callback pointing save-path at `/scratch/ckpt`, triggers async upload — the path that upgrades "best-effort restart" → "guaranteed cross-node resume"). Per-job SSE/gRPC event stream + signed (HMAC) webhooks, at-least-once with 24 h backoff, event types `job.queued…usage.record.sealed` (§32.7).
- **Prereqs.** C-1, C-2; checkpoint/resume at scale (C-13).
- **Done when.** `superheat run --interruptible … -- python train.py` submits, attaches logs, survives a preemption as a `job.preempted`→`job.resuming`→`job.running` sequence with a stable endpoint; webhook signature verifies.
- **Design-doc §.** `platform.md` §33; `connectivity-and-data.md` §B4.1.

---

### Workstream 2 — Direct / reserved demand (own demand + concentration dilution)

> Owner: Backend (yield router) + enterprise sales-engineering. Implements `platform.md` §14, §16–§18. Reserved demand is the rating fuel — contracted GPU-hours move the ABS coupon down and dilute marketplace concentration (`financial-and-roadmap.md` §C4.1, R6).

#### C-5. Wire `direct` as a zero-fee connector into the existing router
- **Action.** Implement `source="direct"` as a `MarketplaceConnector` (`platform.md` §13.1) with `platform_fee_frac=0`, so the router (§16) treats direct as just another source — one whose `$0.40` gross becomes ~$0.40 net vs ~$0.29 on a 27%-fee marketplace (§14). `direct` is **exempt** from the concentration guard (§16.3). Feed real `p_fill` from the direct pipeline so EV rises as the pipeline thickens (the §16.5 worked example: direct overtakes vast.ai at the same node-hour with no code change).
- **Prereqs.** Yield router v1 (Phase B); Renter API (C-1).
- **Done when.** A direct job and a marketplace job compete in one `route_node_hour` call; the router picks max-EV net price; direct revenue does **not** count toward any source's 60% concentration share.
- **Design-doc §.** `platform.md` §14, §16.2–§16.5.

#### C-6. Reserved-instance contracts + the reserved-floor planner
- **Action.** Implement `tenant_class="reserved"`: a customer pre-commits N GPU-hours/mo at a contracted floor (e.g. ~$0.49/GPU-hr, non-interruptible, `reliability_tier_required ≥ standard`, `data-contracts.md` §4). Build the **nightly reserved-floor planner** (`platform.md` §17): forecast each contract's monthly commitment against forecasted thermal windows, pre-allocate high-reliability node-hours spread across the month (not front-loaded), release unused pre-allocations back to spot (reserved is a floor, never a ceiling). `route_node_hour` **step 0** places a pending reserved job unconditionally via `fits()` before the spot auction runs (§16.3).
- **Prereqs.** C-5; thermal-window forecast (§6); premium-tier nodes (Phase B reliability tiering).
- **Done when.** A reserved contract's monthly GPU-hours are met from `premium`/`standard` nodes spread across the month; an over-light reserved hour releases capacity to spot; a fleeting spot spike never causes a missed commitment.
- **Design-doc §.** `platform.md` §14, §17, §16.4; `data-contracts.md` §2, §7.

#### C-7. Concentration guard hardening + contracted-revenue covenant reporting
- **Action.** Confirm the two-layer guard (`platform.md` §16.3): soft graduated penalty from `warn_share` (0.50) → hard covenant stop at `max_share` (0.60), with the only exception being "place + flag" when blocking would idle the node. Surface, per period, the **contracted-revenue ratio** (reserved + HSA + DR ÷ total) and per-marketplace share to a covenant monitor. Add io.net + Akash DePIN connectors *behind* token-FX reconciliation (`platform.md` §18, §19 Phase C) to broaden source diversity.
- **Prereqs.** C-5, C-6; settlement reconciliation (Phase B §18); DR enrollment (C-19).
- **Done when.** No source can exceed 60% trailing-30d revenue without a logged exception; the contracted-revenue ratio is reported per period and trends upward as reserved/DR ramp.
- **Design-doc §.** `platform.md` §16.3, §18.3; `financial-and-roadmap.md` R6, §C4.4.

---

### Workstream 3 — Regional content-addressed cache + data layer (the relay-cost lever)

> Owner: Storage/CDN engineers. Implements `connectivity-and-data.md` Part B §B1 and §A6. This is the workstream that *demonstrates* the relay-cost reduction exit criterion (R2 OPEX line, `financial-and-roadmap.md` §B2.2).

#### C-8. Multi-region PoP cache fan-out (content-addressed blob store)
- **Action.** Deploy regional PoP caches (10–100 TB NVMe, 1–10 Gbps backhaul, dual-stack v4/v6) per major geo. Every artifact keyed `cas:sha256:<digest>`; names (`meta-llama/Llama-3.1-8B`, `pytorch/2.4-cuda12.4`) are manifests = ordered blob-digest lists (`connectivity-and-data.md` §B1.1). Fleet-wide dedup: one copy per tier serves thousands of jobs; integrity-by-rehash on arrival. PoP-miss → PoP (not the node) pulls origin once and caches (§B1.4).
- **Prereqs.** Phase A single-region PoP; node-region sharding (`platform.md` §9).
- **Done when.** A weight blob pulled by one node in a region serves all subsequent nodes in that region from PoP NVMe (one origin pull/region); a corrupted residential transfer is detected, not used.
- **Design-doc §.** `connectivity-and-data.md` §B1.1–§B1.4; §A9 (IPv6 dual-stack).

#### C-9. On-node NVMe cache: LRU + pinning + checkpoint reserve + prefetch
- **Action.** On each node's ≥1 TB/GPU scratch: LRU eviction with **pinning** of the running job's working set; a fixed reserve slice (~150 GB) for checkpoint scratch so a checkpoint always writes locally even when the cache is full (fail-safe, `connectivity-and-data.md` §B1.3). On tentative scheduler assignment, issue a **prefetch hint** so the node pulls manifest blobs down-link during the previous job's tail / pre-heat window — GPU starts computing within seconds, not after a multi-minute pull.
- **Prereqs.** C-8; scheduler pre-warm hint (`platform.md` §6.2 pre-heat coupling).
- **Done when.** GPU compute starts within seconds of job start on a cache hit; checkpoint write never blocked by a full cache; data I/O never blocks the heater/watchdog (lowest-priority local consumer).
- **Design-doc §.** `connectivity-and-data.md` §B1.2–§B1.3; §B0.

#### C-10. Origin pre-warming on the demand stream
- **Action.** Build a control-plane job that watches the demand stream and pre-pulls *trending* artifacts (e.g. a newly released model) into every active region on PoP backhaul **before** the demand wave — one origin pull per region amortized over thousands of node pulls (`connectivity-and-data.md` §B1.3). PoP eviction = weighted LRU-K (frequency × recency × size), biased to keep small hot blobs (popular base layers, top-20 open models) resident, evict large cold datasets first.
- **Prereqs.** C-8; demand-stream visibility (yield router).
- **Done when.** A trending model is resident in all active regions before its first job lands; cache hit-rate metric rises as pre-warming matures.
- **Design-doc §.** `connectivity-and-data.md` §B1.3.

#### C-11. Relay-cost instrumentation + bulk-off-relay enforcement (the R2 demonstration)
- **Action.** Instrument relay-bytes vs direct-bytes and **relay-$/GPU-hr** per node, fed from the per-node bandwidth schema (`connectivity-and-data.md` §B3.2, `path.relay_fallback_active`). Enforce the central contract: bulk (weights/datasets/results/checkpoints) rides PoP down-link / direct mesh, **never** the relay — so the relay carries only KB/s control traffic, collapsing the `B` term in the §A6.1 cost model. Drive relayed-session fraction `r` below ~15–20% via IPv6-first traversal (C-12), keepalives, NAT-aware retries. Any relay-budget alarm throttles data I/O first, never control/heartbeat (§B3.4).
- **Prereqs.** C-8, C-9; multi-region DERP POPs (C-12).
- **Done when.** A before/after report shows relay-$/GPU-hr dropping by ~an order of magnitude vs the Phase-A bulk-on-relay baseline (the $54.4K/mo→residual collapse, `connectivity-and-data.md` §A6.1); `r` is tracked and trending down. **This is exit criterion 4.**
- **Design-doc §.** `connectivity-and-data.md` §A6, §B2.2, §B3.4; `financial-and-roadmap.md` §B2.2 (R2 OPEX line).

#### C-12. Multi-region DERP POPs + IPv6-first traversal + OverlayProvider contingency
- **Action.** Stand up one DERP/relay POP per major geo (lowest-RTT selection, in-region egress); make coordinator, DERP, and PoP caches all dual-stack so the client prefers IPv6 candidates (no NAT on v6 — the single highest-leverage CGNAT mitigation, `connectivity-and-data.md` §A4.2, §A9). Automated key rotation (make-before-break) + instant revocation at fleet scale (§A8). Validate the `OverlayProvider` abstraction against **Nebula** as the documented contingency if headscale's single-coordinator scaling hurts beyond ~10k nodes (§A3.2, §C1 Phase C).
- **Prereqs.** Phase A/B headscale + one DERP region; `OverlayProvider` interface (`node.md`).
- **Done when.** High-v6 regions show measurably lower relay fractions; key rotation/revocation runs unattended fleet-wide within one push cycle; an `OverlayProvider` swap to Nebula passes the agent's job/console contract tests.
- **Design-doc §.** `connectivity-and-data.md` §A4.2, §A8, §A9, §C1.

---

### Workstream 4 — Checkpoint/resume at scale + bandwidth-aware placement

> Owner: Network-placement engineer + storage. Implements `connectivity-and-data.md` §B2–§B3. Makes the interruptible/spot tier safe to sell at fleet scale and keeps data-heavy jobs off slow uplinks.

#### C-13. Checkpoint/resume at scale (RTO/RPO, three-stage, delta-dedup)
- **Action.** Scale the three-stage checkpoint (`connectivity-and-data.md` §B2.2): Stage 1 write local NVMe (seconds, always succeeds, fail-safe), Stage 2 async **delta** upload to PoP (content-defined chunking ~4 MB, ship only changed chunks — a 40 GB state changing ~2 GB ships 2 GB), Stage 3 register durability in control plane. A job is "safely preemptible up to step n" **only** after Stage 3 for n. Adaptive cadence keyed off the thermal coordinator's expected-remaining-window (stable node → 15–30 min; near window-end / imminent yield-preempt → checkpoint now). `T_GRACE_S=30` flushes the in-flight delta; durable resume source is always the last PoP-persisted checkpoint (`data-contracts.md` §7).
- **Prereqs.** Checkpoint/resume v1 (Phase B); C-8 (PoP), C-9 (NVMe reserve).
- **Done when.** Fleet-wide: preempted job resumes useful compute within RTO (p50 ≤ 2 min, p95 ≤ 10 min for ≤40 GB working sets, given the checkpoint is PoP-durable); RPO ≤ one interval (default ≤ 15 min); `superheat-ckpt` jobs achieve guaranteed cross-node resume.
- **Design-doc §.** `connectivity-and-data.md` §B2.1–§B2.5; `platform.md` §26, §34.2.

#### C-14. Bandwidth-aware placement (per-node bandwidth schema → scheduler)
- **Action.** Wire the per-node bandwidth schema (`connectivity-and-data.md` §B3.2 — `up_mbps_floor_24h`, `max_state_gb_for_rto`, `bandwidth_tier A/B/C/D`, `data_heavy_ok`, `egress_cost_flag`) into the scheduler's scoring. Apply placement rules (§B3.3): (1) co-locate data+compute (cache-affinity / same-region PoP); (2) match working-set size to uplink (big-checkpoint jobs → A-tier; LoRA/inference → anywhere); (3) route bulk over direct mesh, never relay — never place a data-heavy job on a relay-only node if a direct node exists, and let the yield router subtract expected relay cost from a relayed node's $/GPU-hr (`egress_cost_flag`); (4) rate-limit result/checkpoint egress below `up_mbps_floor_24h` so it never saturates the homeowner link.
- **Prereqs.** C-11 (relay instrumentation); reliability score (Phase B §4); bandwidth measurement (§B3.1).
- **Done when.** A 200 GB-dataset job is never placed where it would stream cross-region; data-heavy jobs land only on A-tier-uplink nodes; relay cost is honestly subtracted from relayed-node EV so economics stay correct.
- **Design-doc §.** `connectivity-and-data.md` §B3.1–§B3.4; `platform.md` §16.2 (`expected_preempt_cost`).

---

### Workstream 5 — VPP / Demand-Response at fleet scale (the seasonal floor)

> Owner: Backend (utilization engine L4) + partnerships. Implements `platform.md` §25 (L4) and `operations.md` VPP/DR. This is the summer/warm-climate floor that lifts stress-case NCADS (`financial-and-roadmap.md` §B5).

#### C-15. Fleet-wide DR/VPP enrollment (EnergyHub / Renew Home / Voltus / Leap)
- **Action.** Complete enrollment of the fleet's flexible load into DR/VPP programs (`platform.md` §25 L4). EnergyHub already aggregates Rheem water heaters across 30+ utilities — integration is effectively pre-built for our device class (`platform.md` §25). Target $20–100/unit-yr of utility-grade contracted income (`financial-and-roadmap.md` §B2/§B5).
- **Prereqs.** Phase B DR/VPP enrollment *begun*; thermal coordinator (§6); per-region grid-program map (`Superheat_Partnership_and_Capital_Map.md` §6).
- **Done when.** Enrolled flexible-load capacity and DR $/unit-yr are reported per region; DR enrollment counts toward the contracted-revenue ratio (C-7).
- **Design-doc §.** `platform.md` §25 (L4); `financial-and-roadmap.md` §B2.1 line, §B5.

#### C-16. DR event handling in the control loop (charge/shed, never fights comfort or L3)
- **Action.** Implement DR-event handling in the per-node control loop (`platform.md` §29): a DR **charge** event (utility pays to soak grid energy) = a free pre-heat opportunity; a DR **shed** event (utility pays to drop load) = GPU-off, which is itself the comfort-safe default. DR only claims hours the upper layers (L1–L3) declined and comfort always preempts (`platform.md` §25, §29 step 2). Reconcile any **firm** shed obligations against running `premium` L3 SLAs *before* enrolling premium nodes in firm programs (`platform.md` §30.2 open question; `operations.md` DR-vs-comfort-vs-compute).
- **Prereqs.** C-15; utilization control loop (Phase B §29); L3 reserved SLAs (C-6).
- **Done when.** A DR shed never interrupts a comfort obligation or a firm L3 SLA; a DR charge event is monetized as a pre-heat; `IDLE_AND_COAST` hours fall as DR absorbs them (summer/warm-climate floor demonstrated in the §30 duty-cycle example).
- **Design-doc §.** `platform.md` §25, §29, §30; `operations.md` §16 (DR drills draw on chaos rig).

---

### Workstream 6 — Trusted-host confidentiality tier (sizes the R1 TAM ceiling)

> Owner: Security engineer. Implements `security-and-compliance.md` §4.3–§4.4. A *contractual + physical + (future) hardware* construct, not a software trick. Priced and tracked separately on the tape.

#### C-17. Trusted-host vetting + operator agreement + sealed-unit controls
- **Action.** Define and execute trusted-host vetting (`security-and-compliance.md` §4.3): a commercial site under a Superheat operator agreement with controlled physical access, tamper-evident enclosure, **no host admin on the node**, and (where hardware permits) an attestable GPU TEE. Encode the vetting state + operator-agreement artifact as gating before a node can be labeled `trusted_host`.
- **Prereqs.** Commercial-site pipeline (enterprise sales); attestation (Phase A/B §6).
- **Done when.** A site cannot be flagged `trusted_host` without a signed operator agreement + tamper-evident enclosure + no-host-admin attestation on file.
- **Design-doc §.** `security-and-compliance.md` §4.3; §2.3 (residuals it does/doesn't close).

#### C-18. `confidentiality_class` as a first-class registry + scheduler + tape attribute
- **Action.** Carry `confidentiality_class ∈ {standard, trusted_host}` (`data-contracts.md` §2.2) on every node (registry), every job (`trust.confidentiality_required`), and every metered line (`platform.md`). It is **orthogonal to `reliability_tier`** — `trusted_host` must never appear in `reliability_tier_required`. Enforce via `fits()` (`data-contracts.md` §7): a `trusted_host` job lands **only** on a `trusted_host` node, else stays `queued` / `409` (never silently downgraded — C-3). The data layer tags every artifact/volume with its tier so sensitive data never lands on an untrusted node (`connectivity-and-data.md` §B5).
- **Prereqs.** C-17; `fits()` predicate (`data-contracts.md` §7); renter API confidentiality block (C-3).
- **Done when.** A `trusted_host` request is never placed on a standard node (conformance test, `operations.md` §8.3); `trusted_host` revenue is metered separately and poolable as a distinct sub-pool (`security-and-compliance.md` §7.3).
- **Design-doc §.** `security-and-compliance.md` §4.3, §7.3; `data-contracts.md` §2.2, §7.

#### C-19. Size the trusted-host TAM + revenue (the R1 experiment)
- **Action.** Run the R1 experiment (`financial-and-roadmap.md` R1): measure addressable demand that *requires* hardware TEE vs not; size `trusted_host`-tier revenue and utilization; set the target % of fleet that must be TEE-capable hardware; track 4090-successor / data-center-GPU TEE roadmaps for the refresh cycle (`security-and-compliance.md` §4.4).
- **Prereqs.** C-18 (separate metering); direct-demand pipeline (C-5).
- **Done when.** A trusted-host TAM + revenue/utilization report exists with a decided fleet-% TEE target — bounding the R1 confidentiality TAM ceiling.
- **Design-doc §.** `financial-and-roadmap.md` R1, D1; `security-and-compliance.md` §4.4.

---

### Workstream 7 — Renter UX (data-center ergonomics on a basement GPU)

> Owner: Product + frontend + storage. Implements `connectivity-and-data.md` §B4 and `platform.md` §34–§35. Smooth one-command entry, stable sessions across resume, clear eviction.

#### C-20. One-command / SSH / container entry over the overlay (stable across resume)
- **Action.** Deliver `superheat run --gpu RTX4090 --image … --interruptible -- python train.py` and `superheat ssh <job>` as an **outbound reverse tunnel** terminated at a broker (no inbound port on the home router, `connectivity-and-data.md` §B4.1). On preempt→resume the broker re-points the tunnel to the new node so the renter's terminal/endpoint URL is **stable** across a migration (`platform.md` §34.2) — preemption looks like a brief stall, not a failure. Surface tier + confidentiality explicitly on every submit (no silent downgrade).
- **Prereqs.** C-4 (SDK/CLI); C-13 (resume); session subnet + userspace proxy (Phase B §A7).
- **Done when.** A renter `ssh`'d into a spot job keeps a stable session URL across an actual preemption+resume; the GPU "looks like a normal box" (no exposure that it's residential behind CGNAT).
- **Design-doc §.** `connectivity-and-data.md` §B4.1, §A7.3; `platform.md` §34.2.

#### C-21. Persistent volumes with clear, published eviction semantics
- **Action.** Implement the four storage classes with published, configurable TTLs (`connectivity-and-data.md` §B4.2, `platform.md` §34.3): ephemeral scratch (wiped on node change); checkpoint dir (`T_ckpt=24 h`); persistent volume (`--volume`, PoP-backed, re-attached on resume, `T_vol=7 d` after last detach with `volume.gc_warning` at 48 h + 6 h); result/output (`T_out=72 h`). `superheat volume ls` shows `GC-IN` so eviction is never a surprise. GC = per-job key destruction + block TRIM, logged to the audit trail (the privacy backstop, `connectivity-and-data.md` §B5).
- **Prereqs.** C-8 (PoP-backed volumes); per-job key lifecycle (`security-and-compliance.md` §4.2).
- **Done when.** A persistent volume re-attaches on cross-node resume; `volume.gc_warning` fires at 48 h/6 h; GC destroys the per-job key (cryptographic erase) and TRIMs blocks, audit-logged.
- **Design-doc §.** `connectivity-and-data.md` §B4.2, §B5; `platform.md` §34.3.

#### C-22. Result egress + bandwidth-honest UX
- **Action.** Implement `result_egress` modes (`platform.md` §35.2, `connectivity-and-data.md` §B4.3): `renter_bucket` (preferred — scoped write-only creds, node pushes direct-mesh to renter S3/GCS, rate-limited below `up_mbps_floor_24h`, keeps outputs off our infra) and `pop_staging` (`T_out`, pulled via `superheat cp` over fast PoP down-link). Expose `job.placement.bandwidth_tier` so a renter requesting huge egress on a `spot` job sees the bandwidth reality. Per-tier SLO/SLA table surfaced (`platform.md` §34.4): `premium` SLA-backed, `standard`/`spot` best-effort with documented RTO/RPO.
- **Prereqs.** C-14 (bandwidth-aware placement); C-4 (CLI `cp`).
- **Done when.** A big-output `data_heavy` job is steered to an A-tier uplink node; result egress never saturates the homeowner link; the renter can observe `bandwidth_tier` before being surprised by slow egress.
- **Design-doc §.** `platform.md` §35.2, §34.4; `connectivity-and-data.md` §B4.3, §B3.3.

---

### Workstream 8 — Backup-servicer drill (the deal-clearing R7 cure)

> Owner: SRE + finance-ops + ABS counsel. Implements `operations.md` §6 + §5 (DR) and `security-and-compliance.md` §7.1. This is the executed+tested backup-servicer that the rating committee requires (exit criterion 3, R7).

#### C-23. Assemble the escrow package (runbooks + keys + IaC + installer network)
- **Action.** Deposit with the escrow agent the full package (`operations.md` §6.2): the ops runbooks (`operations.md` Part A *is* the human-readable half of the escrow deliverable) + the design docs (`data-contracts.md`, `platform.md`, `node.md`, `connectivity-and-data.md`, `financial-and-roadmap.md`); control-plane signing/counter-signing key custody, overlay-coordination admin, registry/ledger DB access, OTA signing keys; infrastructure-as-code for the control plane; and recognition that the certified installer/T3 network is itself a backup-servicer asset. Released to the named backup servicer on a defined servicing-transfer trigger (Superheat default/insolvency).
- **Prereqs.** Phase B backup-servicer runbook *drafted*; least-privilege RBAC + IaC (`security-and-compliance.md` §7.1).
- **Done when.** Escrow agent holds the complete package; a named backup-servicer agreement is executed; release trigger + key-release mechanics are documented.
- **Design-doc §.** `operations.md` §6.1–§6.2; `security-and-compliance.md` §7.1; `financial-and-roadmap.md` R7.

#### C-24. Run the servicer-failover drill / game-day (before inaugural ABS)
- **Action.** Execute the backup-servicer failover game-day (`operations.md` §5 DR drills, §6.2 item 4): a third party stands up the control plane from escrow within RTO and executes the "first 72 hours" priority order — (a) restore attestation + ingest + scheduler so the fleet earns (Runbook 4.4); (b) verify tape reconciliation + the two-budget telemetry/ledger separation (Runbook 4.5); (c) confirm the safety floor (every node still falls back to heater-only locally regardless of cloud state). Draw on the chaos/fault-injection capability (`operations.md` §16) for the correlated-outage and tape-poisoning scenarios.
- **Prereqs.** C-23; quarterly region-failover game-day capability (`operations.md` §5).
- **Done when.** A failover-drill record proves a third party stood up the plane from escrow within RTO, the fleet resumed earning, the tape reconciled, and no node ever depended on the servicer to stay safe (servicing gap = earning gap, never a safety gap). **This is exit criterion 3.**
- **Design-doc §.** `operations.md` §5, §6.2; `financial-and-roadmap.md` R7, §C4.4.

---

### Workstream 9 — Collateral scale, seasoning & covenant reporting (the ABS-readiness backbone)

> Owner: Structured-finance-systems engineer + ledger eng. Cross-cuts all workstreams; produces the financial artifacts the inaugural ABS underwrites (`financial-and-roadmap.md` §C4.4, §B6).

#### C-25. Drive ≥10,000 seasoned units into the financeable pool
- **Action.** Scale field-ops + onboarding (commercial-first, the financeable class) so ≥10,000 units reach `lifecycle_state='active'` AND `seasoned=true` (≥30 d clean history per node, but the pool needs ≥12-mo metered history, `data-contracts.md` §3, `platform.md` §4.2). The tape (`securitization_tape`) joins on `seasoned=true` and `lifecycle_state='active'` only (`platform.md` §7.3). Tag every node/GPU-hour with its `tranche_id` from day one so issuance pulls a clean static pool without a full-fleet scan (`platform.md` §9, §C1.5).
- **Prereqs.** Field-ops at scale (`operations.md` Part C); reliability seasoning (Phase B §4.2).
- **Done when.** ≥10,000 units carry ≥12-mo metered history and appear on the tape with tranche tags. **This is exit criterion 1.**
- **Design-doc §.** `platform.md` §4.2, §7.3, §9; `financial-and-roadmap.md` §C4.4 (econ §3.5 pool).

#### C-26. Extend the tape to net-of-OPEX + refresh-reserve (covenant-grade reporting)
- **Action.** Extend `settlement` + `securitization_tape` (`financial-and-roadmap.md` §B6.1) with `opex_allocated_usd` (connectivity/cache/cloud by metered GPU-hours; field/insurance/payout per-node flat) and `refresh_reserve_usd` (trapped before residual). `electricity_offset_usd` already exists. This lets DSCR be measured on **net-of-OPEX** (the covenant basis, ~0.5× below net-to-stream) and proves the refresh reserve is funded from cash flow — the novel covenant that clears data-center-ABS methodologies (R4).
- **Prereqs.** Authoritative ledger (Phase B §7); metered OPEX actuals replacing `[A]` assumptions (`financial-and-roadmap.md` §B7.1).
- **Done when.** The tape reports per-node net-of-OPEX and a funded refresh-reserve accrual; the materialized view is idempotently re-runnable from append-only `usage_record` (audit guarantee, `operations.md` §14.1).
- **Design-doc §.** `financial-and-roadmap.md` §B6.1–§B6.3, R4; `platform.md` §7.3–§7.5.

#### C-27. Rising contracted-revenue share trend (the inaugural-ABS covenant)
- **Action.** From C-6 (reserved) + C-7 (covenant monitor) + C-15 (DR) + HSA min-draw, report the **contracted-revenue ratio rising over the deal's life** to satisfy the collateral covenant (`financial-and-roadmap.md` §C4.4, econ §3.5, §3.7). The contracted fixed legs lift the *stress* DSCR floor above the 1.5× turbo-amortization trigger (`financial-and-roadmap.md` §B4.3, §B5.2).
- **Prereqs.** C-6, C-7, C-15.
- **Done when.** The contracted-revenue ratio is reported per period and trends upward; stress-DSCR modeling shows the contracted floor lifting NCADS above the turbo trigger. **This is exit criterion 2.**
- **Design-doc §.** `financial-and-roadmap.md` §C4.4, §B4.3, §B5.2; `platform.md` §16.3, §18.3.

#### C-28. Per-region, seasonal-weighted overlap fed to the tape (honest underwriting)
- **Action.** Operate the per-region, per-month overlap model (HDD-driven space-heat + occupancy-driven DHW, `platform.md` §28.3) and feed the **seasonal-weighted** overlap to the tape — never the winter peak — so residential/warm-climate pools are not over-underwritten (`financial-and-roadmap.md` §B5, R5). 60%×70% is a *commercial* number; blended fleets land at 0.31–0.36 (`platform.md` §22).
- **Prereqs.** Phase A/B telemetry (warm-tier seasonality); thermal coordinator forecaster (§20.4).
- **Done when.** The tape's utilization/overlap inputs are regional + seasonal-weighted; underwriting uses the fleet-mix-weighted number, not the commercial single-segment figure.
- **Design-doc §.** `platform.md` §22, §28.3; `financial-and-roadmap.md` §B5, R5.

---

### Workstream 10 — Cross-cutting hardening (modular refresh, residency, fleet-scale validation)

#### C-29. Year-5 modular GPU-refresh tooling stood up ahead of the covenant (R4)
- **Action.** Field-test + stand up modular GPU-module swap + version-aware re-registration (`financial-and-roadmap.md` R4; `node.md`; `platform.md` registry): retired `gpu` rows keep `retired_at` for tape lineage and must not block a fresh card in the same slot (`gpu_one_live_per_slot` partial index, `platform.md` §1.1). A swap re-provisions a *new* node identity (`connectivity-and-data.md` §A8.3). Confirm the swapped config stays inside the UL/ETL-listed envelope or re-evaluate (`security-and-compliance.md` §10.5).
- **Prereqs.** Refresh-reserve accrual on tape (C-26); modular hardware spec (`node.md`).
- **Done when.** A pilot GPU-module swap + re-registration succeeds with tape lineage preserved and identity re-provisioned; golden test covers "GPU module-swap mid-period, old serial retained" (`operations.md` §14.2).
- **Design-doc §.** `financial-and-roadmap.md` R4; `platform.md` §1.1; `connectivity-and-data.md` §A8.3.

#### C-30. Data-residency as a scheduler placement predicate
- **Action.** Enforce `data_residency_required` as a placement predicate matching `node.region` (`security-and-compliance.md` §12.3): an EU-only job never routes to a US basement; cross-border compute = cross-border data transfer, surfaced not silent. Privacy review / DPIA before EU/CCPA-state entry (R14); occupancy-inference minimization on the thermal forecaster (`security-and-compliance.md` §12.1).
- **Prereqs.** Region-aware scheduler (`platform.md` §9); `fits()` extension.
- **Done when.** A residency-constrained job is rejected from out-of-region nodes; DPIA completed for each privacy-law region entered.
- **Design-doc §.** `security-and-compliance.md` §12.1–§12.4; `financial-and-roadmap.md` R14.

#### C-31. Fleet-scale validation of the Phase-C control surface (sim + chaos + tape)
- **Action.** Extend the discrete-event simulator + chaos suite (`operations.md` §9, §16) to Phase-C breadth: multi-region cache fan-out, preemption storms with PoP-durable resume, relay-flap → data-heavy steering, reserved-floor honoring under load, tape-poisoning rejection, correlated-outage `cloud_degraded` exclusion. All assert the Part A SLO catalog (S1–S20) and the three-way reconciliation harness (§14.3). HIL safety suite + zero-hot-water release gate remain blocking invariants (`operations.md` §12.2, §13.2).
- **Prereqs.** C-13, C-14, C-6, C-26; Phase B simulator.
- **Done when.** sim-nightly + chaos green for all Phase-C scenarios; zero comfort violations; placement p95 < 2 s; idle-leak < 1%; concentration ≤ 60%; three-way reconciliation produces exact orphan/missing/variance buckets with no false positives on clean data.
- **Design-doc §.** `operations.md` §9, §12.2, §13.2, §14.3, §16.

---

### Phase C Exit Checklist — criterion → verifying step

| # | Exit criterion (`financial-and-roadmap.md` §C4.4) | Verifying step(s) | Evidence artifact |
|---|---|---|---|
| 1 | **≥10,000 seasoned units (≥12-mo metered history) available as collateral** | **C-25** (drive the pool); supported by C-26 (net-of-OPEX tape), C-28 (seasonal overlap), C-29 (refresh tooling) | Tape rows: ≥10k nodes `active` + `seasoned=true`, tranche-tagged, ≥12-mo history |
| 2 | **Inaugural-ABS readiness: rising contracted-revenue share** | **C-27** (rising ratio + stress-DSCR floor); built from C-6 (reserved), C-7 (covenant monitor), C-15 (DR) | Per-period contracted-revenue ratio trending up; stress DSCR ≥ 1.5× turbo trigger with contracted legs |
| 3 | **Executed + tested backup-servicer (R7)** | **C-24** (failover game-day); prereq **C-23** (escrow package) | Failover-drill record: third party stood up the plane from escrow within RTO; tape reconciled; safety floor intact |
| 4 | **Cache-driven relay-cost reduction demonstrated (R2)** | **C-11** (relay-$/GPU-hr before/after); built on C-8/C-9/C-10/C-12 (cache + IPv6 + POPs) | Before/after report: relay-$/GPU-hr down ~1 order of magnitude; `r` < ~15–20% and trending down |

**Supporting (not exit-gating but required for a ratable package):** C-1…C-4 (own-demand surface that captures the marketplace fee), C-5 (zero-fee direct routing), C-17…C-19 (trusted-host tier sizing the R1 TAM), C-20…C-22 (renter UX that retains direct demand), C-26/C-28 (covenant-grade tape), C-30 (residency), C-31 (fleet-scale validation against the SLO catalog). These deliver the *ratable, diversified collateral package + smooth UX* of the Phase-C one-line outcome (`financial-and-roadmap.md` §C4) and unlock rung 3–5 (forward flow → term ABS → programmatic).

**Standing invariants enforced throughout (never traded for a Phase-C feature):** outbound-only (no inbound port, C-12/C-20); fail-safe-to-heater (C-9/C-16/C-24, HIL gate C-31); zero-hot-water release gate (`operations.md` §13.2); `data-contracts.md` is normative (every projection conformance-tested, C-1/C-18/C-31).

---

## Part 5 — Master checklist

One flat rollup of every milestone, for tracking. Each line is a deliverable, not a micro-step — the numbered detail (actions, acceptance, design-doc pointers) lives in the per-phase Parts above. Check a box when its "done when" in the source step is met.

Legend: ⛔ = hard safety/financing gate (cannot proceed past it until met).

---

### 0 · Blocking decisions ([Part 1 §1.1](#11--blocking-decisions-to-make-first))

- [ ] **D1 ⛔ Freeze commercial SKU at 8× RTX 4090** — sets BOM, thermal envelope, tape collateral class
- [ ] **D2 Validate true commercial CAPEX (~$20–22K)** — split BOM into compute-host + thermal; restate waterfall
- [ ] **D3 Secure refurb-4090 supply** (~$1,800 × thousands, acceptable failure/warranty)
- [ ] **D4 Decide financing structure** given post-OPEX stress DSCR ~1.1–1.3× (advance rate / contracted floors)
- [ ] **D5 Host agreement + transfer-on-sale + lien strategy** with counsel
- [ ] **D6 UL/ETL listing + a reference permit** in the pilot jurisdiction
- [ ] **D7 Region / pilot-site selection** (controllable thermal load; commercial first)
- [ ] **D8 Bind insurance stack** (product liability, water/fire/electrical)

### 0 · Foundational setup ([Part 1 §1.2](#12--foundational-setup-to-start-engineering))

- [ ] Team/roles assigned (embedded-Linux, networking, distributed-systems, controls, data/ledger, SRE, field-ops)
- [ ] Monorepo + repos created: control-plane, node-agent, node-os image, infra-as-code
- [ ] Tech stack locked (Flatcar-class immutable OS, headscale/WireGuard, Supabase Postgres, TSDB, container runtime + NVIDIA Container Toolkit, fleet/OTA tool)
- [ ] External accounts: cloud, vast.ai host, Supabase, headscale host, cellular SIM, payments rail, observability
- [ ] Pilot: site secured, 8× refurb-4090 host built ([`node.md`](node.md) BOM), HIL bench rig stood up ([`operations.md`](operations.md))
- [ ] **⛔ "Ready to build" gate:** D1 frozen + pilot hardware + HIL rig

---

### Phase A — Attach for revenue ([Part 2](#part-2--phase-a-attach-for-revenue))

**Node OS + agent**
- [ ] Immutable OS image (A/B partitions, NVIDIA driver + Container Toolkit baked in, signed)
- [ ] A/B update + health-gate + auto-rollback (heater-critical vs compute-optional split)
- [ ] Node agent v1: heartbeat/telemetry, container lifecycle, OTA, watchdog, local job cache
- [ ] Thermal guard (local safety only) + hardware safety interlock; recovery ladder → heater-only fallback
- [ ] Agent emits canonical telemetry frame + signed usage record ([`data-contracts.md`](data-contracts.md) §5, §8)
- [ ] Node identity/attestation baseline (secure element / software-bound)

**Connectivity**
- [ ] Outbound-only WireGuard overlay (headscale) joins nodes with zero inbound ports
- [ ] Relay/TURN fallback for CGNAT; direct-vs-relay measured
- [ ] Remote console + renter session path over the overlay

**Control-plane skeleton**
- [ ] Minimal registry (node/SKU/region/owner/financing tag, per-SKU `usable_thermal_kwh`)
- [ ] Telemetry ingest pipeline + dashboards/alerts
- [ ] OTA orchestrator (canary → cohort → fleet, auto-halt)

**Marketplace attach + metering**
- [ ] vast.ai host connector (jobs run in isolated containers; concentration *measured*)
- [ ] Metering v0 — read-only tape: per-unit GPU-hrs + realized $/hr, reconciled to telemetry

**Safety / release gates**
- [ ] ⛔ HIL safety suite passes (thermal interlock, fail-safe-to-heater)
- [ ] ⛔ A/B rollback proven in production (canary→cohort→fleet)
- [ ] ⛔ Zero-hot-water release-gate checklist enforced on every OTA

**Phase A exit ⛔**
- [ ] ≥1,000 commercial units (or pilot cohort) instrumented
- [ ] 12 consecutive months of clean per-unit tape
- [ ] A/B rollback proven with zero hot-water incidents
- [ ] Marketplace concentration reported and <60%

---

### Phase B — Own the orchestration ([Part 3](#part-3--phase-b-own-the-orchestration))

- [ ] Control-plane **scheduler** (matches demand→nodes via canonical `fits()`; anti-thrash hysteresis)
- [ ] **Thermal coordinator** (tank-as-battery, pre-heat; issues run-windows; comfort inviolable)
- [ ] **Authoritative metering + billing ledger** = the securitization tape (double-signed usage records, corrected per-GPU tape view, period-boundary split)
- [ ] **Reliability scoring + tiering** (canonical score → premium/standard/spot/probation; seasoning gate; cloud-down exclusion)
- [ ] **Yield router v1** (highest net $/GPU-hr; ⛔ hard 60% concentration covenant)
- [ ] **Interruptible backlog filler** (own/partner preemptible work — utilization floor)
- [ ] **Checkpoint/resume v1** (checkpoint to NVMe, async to PoP, resume elsewhere)
- [ ] DR/VPP enrollment started (EnergyHub/Renew Home class)
- [ ] Sim-validated before prod: scheduler p95<2s, thermal control law, 60%×70% + 60%-concentration ([`operations.md`](operations.md))
- [ ] Backup-servicer runbook drafted

**Phase B exit ⛔**
- [ ] Owned ledger reconciles within tolerance (usage records ↔ marketplace settlement ↔ bank receipts)
- [ ] Trailing-6-month DSCR ≥ 2.5× on the candidate pool
- [ ] ≥30% of GPU-hours reserved/contracted
- [ ] Warehouse ≥70% utilized / rating-agency pre-read complete

---

### Phase C — Own the demand & UX ([Part 4](#part-4--phase-c-own-the-demand--ux))

- [ ] **Own/direct enterprise API + SDK/CLI** (external job surface, projection of canonical job)
- [ ] **Reserved-instance contracts** (contracted floors → lift stress DSCR above the 1.5× turbo trigger)
- [ ] **Regional content-addressed cache + pre-staging** (serve weights from PoP, not the slow uplink)
- [ ] **Checkpoint/resume at scale + bandwidth-aware placement** (bulk over direct paths, not paid relay)
- [ ] **Fleet-wide VPP / demand-response** (summer/warm-climate floor)
- [ ] **Trusted-host confidentiality tier** (vetted/TEE-capable sites, priced & tracked separately)
- [ ] **Renter UX** (one-command/SSH/container entry; persistent volumes with clear eviction)
- [ ] **Backup-servicer drill executed and tested** (servicer-failover)
- [ ] Relay-cost reduction demonstrated (cache hit-rate) — controls the R2 opex line

**Phase C exit ⛔**
- [ ] ≥10,000 seasoned units (≥12-mo metered history) available as collateral
- [ ] Contracted-revenue share rising over the deal life
- [ ] Backup-servicer fully executed and tested
- [ ] Inaugural ABS readiness (advance/spread targets per [`financial-and-roadmap.md`](financial-and-roadmap.md))

---

*Detail for every line is in the per-phase Parts above; rationale and schemas are in the eight design docs in this same folder. Re-validate the [Part 1 decisions](#11--blocking-decisions-to-make-first) if any business assumption changes.*
