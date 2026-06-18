# Superheat Workspace

Project workspace for **Superheat** — a distributed GPU compute network built through heating systems. RTX 4090 GPUs are embedded inside water heaters across homes and commercial buildings; the waste heat warms the building while the compute is rented out, and the metered revenue is packaged and financed at scale.

> Users already pay to heat buildings. Now the same energy can run AI compute first: **Electricity → AI Compute → Useful Heat.**

## Repository structure

```
superheat-workspace/
├── README.md              ← you are here (workspace index)
├── market-research/       business case: pitch, GPU pricing, economics, securitization, partners
└── gpu-system-design/     technical/operational architecture for running the fleet
```

- **[`market-research/`](market-research/)** — the *why* and the *business*: market pricing research (RTX 4090 ≈ $0.40/hr, H100 trajectory), recalculated unit economics, the compute-backed-securities debt engine, and the partnership/capital map.
- **[`gpu-system-design/`](gpu-system-design/)** — the *how*: the system that turns thousands of unreliable, NAT'd, heat-constrained single-GPU boxes into one standardized, near-zero-idle, sellable compute fleet with a metered ledger clean enough to securitize.

---

## GPU System Design

**Start here:** [`gpu-system-design/system-design-overview.md`](gpu-system-design/system-design-overview.md) — the master overview. It states the five design principles, the layered architecture, and maps the six founding questions to the layers. A rendered diagram is in [`gpu-system-design/architecture-diagram.svg`](gpu-system-design/architecture-diagram.svg).

**Then:** [`gpu-system-design/data-contracts.md`](gpu-system-design/data-contracts.md) — the **NORMATIVE** single source of truth for shared schemas (reliability tiers, the normalized job, the telemetry frame, the signed usage record, thermal constants, feasibility predicates). Every other doc references it rather than redefining.

Each section of the overview expands into its own engineering-grade deep-dive:

| Doc | § | Covers | Questions |
|---|---|---|---|
| [`node-hardware.md`](gpu-system-design/node-hardware.md) | 3 | 4090 node BOM (H1 + commercial), spec-floor, 4090-vs-alternatives, 8-vs-12-GPU decision, thermal/power/network hardware, year-5 modular swap | Q2, Q3 |
| [`node-software-os.md`](gpu-system-design/node-software-os.md) | 4 | Immutable OS + A/B updates, the node agent (state machine, control loop), tenant isolation, recovery ladder, attestation | Q1, Q2 |
| [`connectivity.md`](gpu-system-design/connectivity.md) | 5 | Outbound-only WireGuard overlay, CGNAT traversal, relay-cost model, segmentation, key management, remote console/sessions | Q1, Q6 |
| [`control-plane.md`](gpu-system-design/control-plane.md) | 6 | Registry, telemetry, OTA orchestrator, reliability scoring, scheduler, thermal coordinator, metering ledger → securitization tape | Q1, Q5, Q6 |
| [`demand-routing.md`](gpu-system-design/demand-routing.md) | 7 | Marketplace connectors (vast.ai first), normalized job model, own/direct demand, the yield router + concentration guard, settlement | Q4, Q5 |
| [`utilization-engine.md`](gpu-system-design/utilization-engine.md) | 8 | Tank-as-thermal-battery model, thermal opportunity window, four-layer no-idle stack, reliability tiering, seasonality | Q5, Q6 |
| [`data-io.md`](gpu-system-design/data-io.md) | 9 | Regional content-addressed cache, checkpoint/resume, bandwidth-aware placement, renter UX, data lifecycle | Q6 |
| [`security-trust.md`](gpu-system-design/security-trust.md) | 10 | Three-way trust/threat model, tenant isolation, honest no-TEE limitation + trusted-host tier, attestation, abuse, backup-servicer | cross-cutting |
| [`roadmap-and-risks.md`](gpu-system-design/roadmap-and-risks.md) | 11–12 | Phase A→B→C build roadmap mapped to the financing ladder, and the consolidated risk register | cross-cutting |
| [`capex-reconciliation.md`](gpu-system-design/capex-reconciliation.md) | — | Decision memo: true commercial CAPEX (~$20–22K, not $18K) + 8-vs-12-GPU SKU, with restated payback/IRR/DSCR | underwriting |

**Foundational, financial & operational docs:**

| Doc | Covers |
|---|---|
| [`data-contracts.md`](gpu-system-design/data-contracts.md) | **NORMATIVE** canonical schemas: reliability-tier enum, reliability-score formula, normalized job, telemetry frame, signed usage record, thermal constants, feasibility predicate |
| [`renter-api.md`](gpu-system-design/renter-api.md) | Renter-facing REST/gRPC API + SDK/CLI, job lifecycle, tier/confidentiality disclosure, SLO-per-tier, data I/O, error model |
| [`opex-and-financial-model.md`](gpu-system-design/opex-and-financial-model.md) | Consolidated per-unit/fleet OPEX, corrected declining-stream waterfall, IRR/DSCR with stated financing assumptions, utilization sensitivity |
| [`observability-slo-incident.md`](gpu-system-design/observability-slo-incident.md) | Consolidated SLO catalog, observability stack at 1M nodes, alerting/on-call, incident runbooks, control-plane DR/HA, backup-servicer escrow |
| [`testing-and-simulation.md`](gpu-system-design/testing-and-simulation.md) | Test pyramid + contract conformance, fleet-scale discrete-event simulator, control-system validation, HIL safety rig, chaos, release gates |
| [`field-operations-and-onboarding.md`](gpu-system-design/field-operations-and-onboarding.md) | Installer certification, install workflow, node provisioning/identity bootstrapping, RMA, year-5 swap as a field op, **homeowner-churn / node recovery** (the repossession analog), customer onboarding |
| [`legal-regulatory-compliance.md`](gpu-system-design/legal-regulatory-compliance.md) | Electrical/plumbing code & permitting, UL/ETL listing, insurance/liability, data privacy (GDPR/CCPA), KYC/AML & lawful process, consumer-financing reg, tax, per-region entry gating |
| [`rental-walkthrough.md`](gpu-system-design/rental-walkthrough.md) | "Day in the life" end-to-end traces (rental, thermal node-day, preemption+resume, node lifecycle) tying every doc together — integrative, non-normative |

### The six founding questions

1. Remote operations & maintenance of scattered single GPU nodes → `connectivity.md`, `node-software-os.md`, `control-plane.md`
2. What OS / software a single node needs → `node-hardware.md`, `node-software-os.md`
3. RTX 4090 config that satisfies the most renters at lowest cost → `node-hardware.md`
4. Auto-dispatch compute + integrate vast.ai-style marketplaces → `demand-routing.md`
5. Build our own network to drive utilization toward zero idle → `demand-routing.md`, `utilization-engine.md`
6. Efficient orchestration + smooth rental & data I/O everywhere → `connectivity.md`, `control-plane.md`, `utilization-engine.md`, `data-io.md`

Q4 and Q5 are answered as **one hybrid strategy**, not a fork: attach to marketplaces first, build our own orchestration in parallel, route every node-hour to the highest-yield source.

### Cross-cutting findings flagged during design

These surfaced while writing the deep-dives and need a decision/experiment (full list in [`gpu-system-design/roadmap-and-risks.md`](gpu-system-design/roadmap-and-risks.md)):

- **Commercial CAPEX is likely ~$20–22K, not $18K** — the memo's $2,500 thermal-BOM line doesn't cover a real 8-GPU host. See [`capex-reconciliation.md`](gpu-system-design/capex-reconciliation.md).
- **8-vs-12-GPU commercial SKU is undefined** — pitch says 12/33 kW, economics models 8/3.3 kW. Freeze before hardware/BOM/tape.
- **60%×70% utilization is consistent for commercial, optimistic for residential** — warm-season/warm-climate hours are lower; model per region.
- **Consumer 4090s have no robust TEE** — no hardware-enforced confidentiality on standard nodes; needs the trusted-host tier + renter disclosure.
- **CGNAT relay bandwidth is a real opex line** — drives the regional-cache + prefer-direct strategy.

**Recommended build order** ([`roadmap-and-risks.md`](gpu-system-design/roadmap-and-risks.md)): **Phase A** (attach to marketplaces for revenue + tape) → **Phase B** (own orchestration + ledger) → **Phase C** (own demand + data layer + VPP).

---

## Getting Started

```bash
git clone https://github.com/hominluo/superheat-workspace.git
cd superheat-workspace
```

The design documents are Markdown; the architecture diagram is SVG (viewable in any browser or GitHub). No build step yet — this is design-stage.

## Status

Draft v0.1, June 2026. Design documents, not implementation.

## License

TBD
