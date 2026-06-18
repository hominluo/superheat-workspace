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

The design is **8 documents** (consolidated from an earlier 19):

| Doc | Covers | Questions |
|---|---|---|
| [`system-design-overview.md`](gpu-system-design/system-design-overview.md) | Principles, layered architecture, the six founding questions, and the end-to-end walkthroughs (day-in-the-life rental, thermal node-day, preemption/resume, node lifecycle) | all |
| [`data-contracts.md`](gpu-system-design/data-contracts.md) | **NORMATIVE** canonical schemas: reliability-tier enum, reliability-score formula, normalized job, telemetry frame, signed usage record, thermal constants, feasibility predicate | — |
| [`node.md`](gpu-system-design/node.md) | The GPU node — hardware (4090 BOM, spec-floor, 8-vs-12-GPU decision, thermal/power/network, year-5 swap) **and** software/OS (immutable Linux + A/B, the agent, isolation, attestation, recovery ladder) | Q2, Q3, Q1 |
| [`connectivity-and-data.md`](gpu-system-design/connectivity-and-data.md) | Outbound-only WireGuard overlay (CGNAT traversal, relay-cost model, key mgmt) **and** data I/O (regional cache, checkpoint/resume, bandwidth-aware placement, renter data UX) | Q1, Q6 |
| [`platform.md`](gpu-system-design/platform.md) | The cloud platform: control plane (registry, telemetry, OTA, reliability scoring, scheduler, thermal coordinator, metering ledger→tape), demand & yield routing, utilization engine, and the renter API/SDK | Q1, Q4, Q5, Q6 |
| [`security-and-compliance.md`](gpu-system-design/security-and-compliance.md) | Three-way trust/threat model, isolation, honest no-TEE limitation + trusted-host tier, attestation, abuse/ToS, backup-servicer — **and** legal/regulatory: code & permitting, UL/ETL, insurance, privacy (GDPR/CCPA), KYC/AML, tax, per-region gating | cross-cutting |
| [`financial-and-roadmap.md`](gpu-system-design/financial-and-roadmap.md) | CAPEX reconciliation (~$20–22K), the OPEX model + corrected declining-stream waterfall (IRR/DSCR with financing assumptions, utilization sensitivity), the Phase A→B→C build roadmap, and the consolidated R1–R15 risk register | underwriting |
| [`operations.md`](gpu-system-design/operations.md) | Observability/SLO catalog + incident runbooks + DR/HA, testing & fleet-scale simulation + release gates, and field ops (installer certification, provisioning, RMA, year-5 swap, **homeowner-churn / node recovery**, onboarding) | cross-cutting |

### The six founding questions

1. Remote operations & maintenance of scattered single GPU nodes → `node.md`, `connectivity-and-data.md`, `platform.md`
2. What OS / software a single node needs → `node.md`
3. RTX 4090 config that satisfies the most renters at lowest cost → `node.md`
4. Auto-dispatch compute + integrate vast.ai-style marketplaces → `platform.md`
5. Build our own network to drive utilization toward zero idle → `platform.md`
6. Efficient orchestration + smooth rental & data I/O everywhere → `connectivity-and-data.md`, `platform.md`

Q4 and Q5 are answered as **one hybrid strategy**, not a fork: attach to marketplaces first, build our own orchestration in parallel, route every node-hour to the highest-yield source.

### Cross-cutting findings flagged during design

These surfaced while writing the design and need a decision/experiment (full list in [`gpu-system-design/financial-and-roadmap.md`](gpu-system-design/financial-and-roadmap.md)):

- **Commercial CAPEX is likely ~$20–22K, not $18K** — the memo's $2,500 thermal-BOM line doesn't cover a real 8-GPU host. See [`financial-and-roadmap.md`](gpu-system-design/financial-and-roadmap.md).
- **8-vs-12-GPU commercial SKU is undefined** — pitch says 12/33 kW, economics models 8/3.3 kW. Freeze before hardware/BOM/tape.
- **60%×70% utilization is consistent for commercial, optimistic for residential** — warm-season/warm-climate hours are lower; model per region.
- **Consumer 4090s have no robust TEE** — no hardware-enforced confidentiality on standard nodes; needs the trusted-host tier + renter disclosure.
- **CGNAT relay bandwidth is a real opex line** — drives the regional-cache + prefer-direct strategy.

**Recommended build order** ([`financial-and-roadmap.md`](gpu-system-design/financial-and-roadmap.md)): **Phase A** (attach to marketplaces for revenue + tape) → **Phase B** (own orchestration + ledger) → **Phase C** (own demand + data layer + VPP).

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
