# Superheat — The GPU Node (Hardware + Software/OS)

**Status:** Draft v0.1 · **Date:** June 2026 · **Audience:** Hardware/manufacturing engineering, ops, underwriting, and software engineering
**Scope:** The complete standardized RTX 4090 compute node that lives inside a Superheat water heater — both its **hardware** (bill of materials, spec floor, GPU selection, the unresolved commercial form-factor, the thermal/electrical/networking interfaces, and the year-5 modular GPU swap) and its **software/OS** (the operating system, the update/rollback machinery, the node agent and its state machine, the local control loop, tenant isolation, the recovery ladder, the thermal-controller interface, the telemetry/usage-record producer, and node identity/attestation).

This document expands §3 ("The GPU node — hardware") and §4 ("The GPU node — software & OS") of `system-design-overview.md` into a single coherent treatment of the node. **Part A** covers the physical node; **Part B** covers the software that runs on it. The two parts share one hardware root of trust, one thermal interface, and one set of per-SKU constants, so they are presented together.

It cross-references `connectivity-and-data.md` (the outbound-only overlay network that reaches the node and the data/checkpoint path), `platform.md` (the cloud control plane / scheduler / utilization engine / renter API that registers, meters, and assigns the node), `security-and-compliance.md` (the trust model and regulatory posture), `financial-and-roadmap.md` (CAPEX reconciliation, the opex/financial model, refurb supply, and the 8-vs-12 decision), `operations.md` (observability, testing, and field operations), and `data-contracts.md` (the normative wire schemas). It stays numerically consistent with `../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md`, `../market-research/GPU_Pricing_Research_4090_H100.md`, and `../market-research/Superheat AI Data Center.md`.

It deliberately does **not** cover the mechanical/plumbing design of the tank, heat exchanger, or coldplate loop. It covers the electronics, the compute BOM, the on-node software, and the sensing/control interface the node needs to be a good thermal citizen.

---

## 0. Design constraints inherited from the five principles

Every decision below — hardware and software alike — is forced by the overview's five principles. The hardware treats them as design requirements; the software treats them as acceptance criteria for every component.

| Principle | Hardware consequence | Software consequence |
|---|---|---|
| **(1) Outbound-only networking** | No need for inbound-capable NIC features, port forwarding, or public-IP hardware. But the link must reliably *dial out*: NIC + (commercial) failover modem are mandatory; relay-friendly path matters more than raw bandwidth. See §10. | Nothing on the node ever listens for inbound connections from the public internet; all control, OTA, and console traffic is node-initiated over the overlay (Part B). |
| **(2) Fail-safe to a water heater** | Compute electronics must be galvanically and logically *separable* from the heating element control. A dead/removed GPU node must leave a working water heater. This drives a two-domain power and control split (§6, §7). | No software state may brick the unit or cut hot water; `HEATER-ONLY SAFE-MODE` is an absorbing state reachable from everywhere (§14.4), and the heater MCU heats even if the agent is gone (§16). |
| **(3) Compute is thermally coupled** | The node must *sense* tank temperature and heat demand and *control* GPU power state. This adds a thermal controller MCU and sensors to the BOM (§7). | The agent treats the tank as a battery, not the GPU as a free runner; the local control loop power-caps the GPU to fit thermal headroom (§14.3). |
| **(4) Measure, then price reliability** | Every node ships with the sensors to self-report power, temp, util, link quality — telemetry is a BOM requirement, not an afterthought (§7.4). | The agent is the source of truth for the telemetry and signed usage records that score and bill the node (§15). |
| **(5) Hybrid demand** | Hardware must be marketplace-generic: a stock NVIDIA driver target, standard PCIe, standard NVMe — nothing that prevents a node from looking like an ordinary 4090 host to Vast.ai/RunPod/Salad. | The agent runs whatever the control plane assigns and is demand-source-agnostic. |

The governing software constraint over all of them: **a node may be physically unreachable for months.** Every mechanism in Part B is designed to recover *without a truck roll*.

Two more constraints come from the financing model in the economics memo:

- **Securitizable standardization.** The fleet must be a small number of identical SKUs so the metering ledger (the securitization tape) describes homogeneous collateral. One H1 SKU, one commercial SKU, a small set of climate variants — no per-install bespoke builds. Whole-image A/B updates reinforce this: a node's identity includes exactly one running image digest.
- **Year-5 modular GPU swap.** The trust funds a "GPU-refresh reserve"; the hardware must make a field GPU module swap cheap and safe (§8), and the software must detect the new card and re-register it (§8.2, §17). The heater shell has a 10-year life; the GPUs have a ~5–6-year revenue life. The node is designed around *two GPU generations per heater*.

---

# Part A — Hardware

## 1. The two standard form factors

| | **H1 (home)** | **Commercial** |
|---|---|---|
| GPUs | 1× RTX 4090 | 8× RTX 4090 (recommended SKU — see §4) |
| GPU power | ~412 W nominal / 450 W design point (per GPU) | ~3.3 kW nominal (8 × 412 W) / up to ~3.6 kW peak (8 × 450 W) — see §6.1 |
| Total system power | ~3.3 kW (heating-system class; GPU + balance) | ~3.6 kW GPU (design point) + ~0.4 kW host ≈ **~4 kW node** (see §6.1) |
| Tenancy | Single-tenant (1 renter at a time) | Sliceable: 8 independent single-GPU rentals **or** 2/4/8-GPU multi-GPU jobs |
| Electrical service | 240 V / 30–40 A residential branch (water-heater circuit) | 240 V commercial; one or two 40 A circuits, or 208 V 3-phase at larger sites |
| Networking | Residential link (Ethernet to home router) | Residential/commercial link **+ cellular failover** (§6) |
| Reliability tier | Spot/interruptible default | High-reliability/premium-capable (see `platform.md`, the utilization engine / reliability tiering) |
| Installed CAPEX | ~$6–9K customer price (financed) | **~$18K** fully loaded |
| Net economics | ~$2.3K/yr DHW-only → ~$8.5K/yr w/ space heat | **~$11.7K/yr** base (60% util × 70% overlap, 25% fee) |

The H1 is a consumer appliance with one card. The commercial unit is a small, thermally-coupled GPU server. They share the **same per-GPU compute spec floor** (§3) and the **same NVIDIA software target** so that Part B ships one image family and the control plane treats a commercial node as "8 logical H1s on one host."

> **Power figures — nominal vs design point.** The per-GPU number appears two ways across the sources and both are kept deliberately: **412 W is the NOMINAL draw** (economics memo, used for the 8 × 412 W ≈ 3.3 kW nominal envelope and the tape's energy accounting) and **450 W is the DESIGN POINT** (pricing research peak/transient, used for PSU/breaker sizing). Sizing always uses 450 W; energy/heat accounting uses the measured wall draw, which sits between the two. See §6.1.

> **Note on the pitch's "Home Unit = 1kW GPU":** Slide 11 of `Superheat AI Data Center.md` assumes a *Blackwell-class ~1 kW GPU* for its 1 GW math. The buildable 2026 fleet uses the **RTX 4090 at ~412 W nominal / 450 W design point**, not a 1 kW card. The 1 GW target is a long-horizon vision; this doc is the shippable 4090 reality. The H1 is built around a **3.3 kW heating system**, so its *thermal* path has ample headroom for more powerful future cards, and it is **swap-ready to a ~550 W 5090-class successor with no electrical changes** (year-5 swap, §8). A **~1 kW-class Blackwell card is a different story**: it would exceed the H1's 1000 W PSU and may exceed the 240 V / 30 A water-heater branch, so it requires a PSU and possibly a circuit upgrade — it is *not* a drop-in. See §9.7. (Earlier drafts overclaimed that the envelope "already supports" a 1 kW card; that is true thermally but not electrically.)

---

## 2. Why RTX 4090 (and why not the alternatives)

The economics memo and pricing research make the 4090 the clear workhorse. Restated as a hardware-procurement decision:

| Card | 2026 acquisition | Rental $/GPU-hr (median) | Power | Verdict for Superheat |
|---|---|---|---|---|
| **RTX 4090** (24 GB) | **~$1,800 refurb** | **$0.40 Vast / $0.44 index** (spot floor $0.15–0.25) | ~412–450 W | **Chosen.** Cheapest training-class card/hr, price-stable near floor, deepest marketplace demand, refurb supply exists, 24 GB covers the mid-model fine-tune/inference/diffusion bulk. |
| RTX 5090 (32 GB) | ~$2,400 | $0.53 | ~550 W | Strong year-5 refresh candidate. Newer silicon → slower decay, fewer cards per kW. But 2026 supply is tighter and pricier; not the launch SKU. The economics memo's "5090 build" (6× @ 550 W) is an equivalent-revenue alternate, **preferred going forward**, not for first deployment. |
| A100 SXM4 (80 GB) | secondary ~$5–10K | $0.91 | 400–500 W (SXM4; ~400 W is the PCIe variant) | Higher $/hr but data-center form factor (SXM, NVLink, 8-GPU baseboards) does not fit a heater, needs real cooling, and the supply is enterprise-channel, not refurb-consumer. Wrong asset class. |
| H100 SXM (80 GB) | $35–40K new / $6–15K used | $2.00–3.12 | ~700 W | Premium silicon, premium heat, premium price — but SXM/HGX baseboards, 700 W/card, and TEE expectations make it a data-center card, not a heater card. Reserve for a future "trusted-host" tier discussion in `financial-and-roadmap.md`, not the fleet. |
| RTX 3090 (24 GB) | ~$700 used | $0.16 | ~350 W | Too close to the floor to finance; the rental rate barely clears host marginal cost. Not a financeable asset. |

**Refurb supply is itself a design assumption.** At $1,800/card the economics work; at the deck's stale $3,500/card they do not. Procurement must validate a sustainable refurb 4090 channel at scale (10,000+ cards) — flagged in `financial-and-roadmap.md` and §9 open questions. The card is a **standard PCIe consumer 4090** (founders or AIB partner board, 2- or 3-slot, 16-pin 12VHPWR). No SXM, no NVLink — the fleet's multi-GPU jobs ride PCIe (§3), which is fine for the targeted workload mix (independent single-GPU rentals dominate; data-parallel fine-tunes tolerate PCIe).

---

## 3. The "don't starve the GPU" per-GPU spec floor

The whole BOM philosophy: spend *exactly enough* host around each 4090 that the GPU is never the idle component for common renter workloads — and not a dollar more, because the heat-offset economics don't reward over-building. Below the floor, the node accepts fewer workloads and earns less $/GPU-hr; far above it, CAPEX rises without revenue. This is the cost/coverage sweet spot.

| Component | Per-GPU floor | Per-GPU target | Rationale |
|---|---|---|---|
| **CPU cores** | 4 physical / GPU | **6 physical / GPU** (≥1 fast core/GPU + headroom) | Data loading, image decode, tokenization, and dataloader workers for fine-tune + inference pipelines. Starving CPU shows up as low GPU util on data-heavy jobs → lower $/hr. |
| **System RAM** | 32 GB / GPU | **48 GB / GPU** | Many renter images assume RAM ≈ 1.5× VRAM (24 GB → ~36 GB). 48 GB clears that heuristic with headroom; 32 GB is the floor (just below 1.5×VRAM but enough for the common case). Below 32 GB popular images OOM. |
| **NVMe scratch** | 1 TB / GPU | **1–2 TB Gen4 NVMe / GPU** | Model + dataset staging, checkpoint/resume I/O (mandatory for interruptible work — see `platform.md` (the utilization engine) and §9 of the overview). Must be fast: checkpoints to local NVMe are what make preemption safe. |
| **PCIe** | Gen4 ×8 / GPU | **Gen4 ×16 / GPU** (×8 acceptable on dense commercial) | Avoid host↔GPU transfer bottleneck on multi-GPU data-parallel jobs. Consumer 4090 is Gen4 ×16; the constraint is lane availability on the motherboard/risers. |
| **Upload bandwidth** | Best available residential; **measured & scored** | symmetric/business link where the site has it | Result/checkpoint egress is the consumer-link bottleneck. This is *scored*, not provisioned — the control plane prices reliability around it (`platform.md`). Commercial sites should target a business link + cellular failover (§6). |
| **VRAM** | 24 GB (the card) | 24 GB | Fixed by the 4090; sets the addressable workload tier (mid-size LLM fine-tune/inference, diffusion, render, batch ML). |

**Reading the table for the two SKUs:**
- **H1 (1 GPU):** 6 cores, 48 GB RAM (floor 32), 1–2 TB NVMe, one ×16 slot.
- **Commercial (8 GPU):** 48 cores, 384 GB RAM (48 GB/GPU; floor 256), 8–16 TB NVMe, 8× ×8–×16 lanes. This is a real workstation/server-class host (§5).

---

## 4. The 8-vs-12 GPU commercial conflict — recommendation

**The conflict.** The pitch (`Superheat AI Data Center.md`, Slide 3 & 11) defines the commercial unit as **12 GPUs / 33 kW**. The economics memo (`Superheat_Recalculated_…`) underwrites **8× RTX 4090 @ 412 W ≈ 3.3 kW nominal** (up to ~3.6 kW at the 450 W design point, §6.1) — an H1-*class* thermal envelope — and *all* financing math (CAPEX, DSCR, ABS pool sizing) rests on the 8-GPU number. The two cannot both be the SKU that drives the BOM and the securitization tape.

**Recommendation: freeze the commercial SKU at 8× RTX 4090 (~3.3 kW GPU).** Treat "12 GPU / 33 kW" as a larger optional configuration that must be *separately re-underwritten* before it can enter a financed pool.

**Why 8, concretely:**

1. **It matches the underwritten economics exactly.** $18K CAPEX = 8 × $1,800 GPU ($14,400) + $2,500 thermal BOM + $1,100 install. ~$11.7K/yr, ~18-mo payback, 45% IRR, DSCR 3.5×/2.0× — every number in the memo and on Slide 8 is the 8-GPU number. Shipping 12 silently breaks the tape.
2. **~3.3 kW nominal (up to ~3.6 kW at the 450 W design point) is a real heating-system thermal envelope.** The heat the tank must absorb is **~3.3 kW nominal / up to ~3.6 kW peak GPU heat** (8 × 412 W nominal → 8 × 450 W design point; see §1, §6.1) — stated honestly at the design point, not the lower nominal number. Even at 3.6 kW peak this stays in the same class as the H1's 3.3 kW heating system, i.e. a single tank can absorb it. 33 kW (12 GPUs the pitch's way, *or* a misread of the kW figure) is a different mechanical, plumbing, and electrical-service problem (3-phase, multiple tanks/large buffer, far higher minimum heat draw to stay thermally overlapped).
3. **8 maps cleanly to commodity host platforms.** An 8-GPU PCIe host (single high-core CPU + 7 PCIe slots via bifurcation/PLX, or a dual-socket board) is a known, buildable, serviceable configuration. 12 PCIe 4090s on one host pushes into exotic riser/PLX topologies, 208 V 3-phase, and >4 kW of host power — more like a mini crypto/render rig than a standardized appliance.
4. **Thermal-overlap risk scales with GPU power.** The 70% thermal-overlap assumption (the share of GPU-hours whose heat is actually wanted) gets *harder* to hit as you raise continuous heat output relative to the host's hot-water/space-heat load. 12 GPUs need a bigger committed thermal draw (a bigger HSA minimum-draw covenant) to stay overlapped — concentrating exactly the risk the economics memo's risk register names.

**BOM / thermal / underwriting implications of the recommendation:**

- **BOM:** Build the 8-GPU host below (§5). Do not design the launch chassis around 12 slots; design it around 8 with clean serviceability. A "12-GPU XL" can reuse the thermal controller, NIC/failover, and OS but needs its own motherboard/PSU/chassis/3-phase study.
- **Thermal:** Size the heat-capture loop and tank-as-battery model for ~3.3 kW nominal / up to ~3.6 kW peak continuous GPU heat (the 450 W design point). The thermal coordinator (`platform.md`, the control plane / §6/§8 of the overview) is tuned for this envelope, importing the per-SKU `usable_thermal_kwh` anchored in §7.
- **Underwriting:** The securitization tape collateral line = "8× 4090 commercial unit, $18K cost basis." If a 12-GPU XL ships, it enters pools only after re-underwriting (new CAPEX, new DSCR, new overlap assumption) and is tagged as a distinct collateral class. This is exactly the "resolve before hardware freeze" item in overview §12 and `financial-and-roadmap.md`.

**If the business insists on 12** (because the 1 GW narrative wants fewer sites): then the entire memo must be re-run at 12× and Slide 8's $18K/$11.7K/DSCR figures restated. That is a deliberate underwriting decision, not a BOM tweak — and it must precede hardware freeze.

---

## 5. Bill of materials

Costs are 2026 estimates, reconciled to the economics memo's $18K commercial total. GPU = $1,800 refurb (memo). Thermal BOM ($2,500) and install ($1,100) are the memo's flagged *assumptions* and are carried verbatim so the totals reconcile; replace with actuals as they firm up. The compute-host line items below are *inside* the memo's structure — see the reconciliation note after the table.

### 5.1 H1 (1× 4090) BOM

| Component | Spec | Est. cost | Notes |
|---|---|---:|---|
| GPU | 1× RTX 4090, 24 GB, PCIe Gen4 ×16, refurb | $1,800 | Field-replaceable module (§8) |
| CPU | 6-core / 12-thread (e.g., Ryzen 5 / Core i5 class) | $150 | Meets per-GPU floor with headroom |
| Motherboard | Consumer ATX/mATX, 1× PCIe ×16, M.2 ×2, IPMI not required | $130 | Must expose fan/temp headers for thermal MCU tie-in |
| RAM | 48 GB DDR5 (e.g. 2×24, or 64 GB 2×32) | $130 | Floor 32 GB; ship 48 to clear 1.5× VRAM (§3) |
| NVMe | 2 TB Gen4 NVMe | $120 | Scratch + checkpoint; OS on a separate small device or partition |
| OS boot media | 128 GB durable NVMe/eMMC (A/B partitions) | $25 | Immutable A/B image (Part B, §11) |
| PSU | 1000 W 80+ Gold (single-rail, 12VHPWR) | $130 | Headroom over ~450 W GPU + ~150 W host |
| NIC | Onboard 2.5 GbE | $0 | Outbound-only; Wi-Fi optional add |
| TPM 2.0 / secure element | Discrete or firmware TPM 2.0 (onboard header) | $5 | **Hardware root of trust:** backs OS attestation (§18) and stores the node identity / usage-record signing key (`data-contracts.md` §5, §8; `security-and-compliance.md`). Required — attestation depends on it. |
| Thermal controller | MCU board: tank-temp + flow sensors, GPU power-state relay, safety interlock | $90 | §7 — the thermal-coupling interface |
| Enclosure / mounting | Sealed compute enclosure integrated into heater chassis | (in thermal BOM) | Heat goes to water, not to room air |
| **Compute host subtotal** | | **~$2,680** | GPU $1,800 + host ~$880 (incl. 48 GB RAM + TPM) |

The H1's *thermal balance-of-system, tank, plumbing, and install* sit in the customer's $6–9K installed price (financed); the compute-host subtotal is the portion that maps to the GPU + a slice of the thermal BOM. The H1 is a single-card consumer appliance, so its BOM is dominated by the card.

### 5.2 Commercial (8× 4090) BOM — reconciled to $18K

| Component | Spec | Est. cost | Notes |
|---|---|---:|---|
| GPUs | 8× RTX 4090, 24 GB, refurb | **$14,400** | 8 × $1,800 — matches memo exactly |
| CPU | 1–2× server/HEDT, **≥48 physical cores total** | (in thermal/host BOM) | 6 cores/GPU floor → 48 cores for 8 GPUs |
| Motherboard | Server/WS board, 7–8× PCIe Gen4 (bifurcation/PLX risers), IPMI/BMC | (in host BOM) | BMC enables out-of-band recovery alongside the agent |
| RAM | **384 GB** DDR4/5 ECC (48 GB/GPU; floor 256) | (in host BOM) | ECC for a multi-tenant always-on host; 48 GB/GPU clears 1.5× VRAM (§3) |
| NVMe | 8–16 TB Gen4 (1–2 TB/GPU, per-tenant scratch) | (in host BOM) | Per-tenant volumes with eviction semantics (overview §9) |
| OS boot media | Mirrored A/B boot devices | (in host BOM) | |
| PSU | 2× redundant 2000–2200 W (or 2×1600 W) 80+ Platinum | (in host BOM) | sized to the 450 W design point: ~3.6 kW GPU + ~0.4 kW host ≈ 4 kW (§6.1); redundancy for premium tier |
| NIC | 2.5/10 GbE primary | (in host BOM) | |
| **Cellular failover modem** | 4G/5G LTE modem + SIM, auto-failover router | (in host BOM) | Mandatory on commercial (§6); raises reliability tier |
| **TPM 2.0 / secure element** | Discrete TPM 2.0 (server board header) or BMC-integrated | (in host BOM) | **Hardware root of trust:** backs OS attestation (§18) and stores the node identity / usage-record signing key (`data-contracts.md` §5, §8; `security-and-compliance.md`). For a multi-tenant always-on host this also underpins the `trusted_host` confidentiality class (`data-contracts.md` §2.2). Required. |
| Thermal controller | MCU: multi-zone tank/flow/ambient sensing, per-GPU power-state control, hardware safety interlock | (in thermal BOM) | §7 |
| Chassis | 8-GPU thermally-coupled enclosure (coldplate/immersion-ready), serviceable GPU modules | (in thermal BOM) | Designed for field GPU swap (§8) |
| Thermal balance-of-system | Tank, heat exchanger, coldplate/immersion loop, pump, power electronics, networking, enclosure, assembly | **$2,500** | Memo assumption — carried verbatim |
| Installation | Plumber + electrician | **$1,100** | Memo assumption |
| **Total installed CAPEX** | | **~$18,000** | $14,400 + $2,500 + $1,100 |

**Reconciliation to the $18K.** The economics memo's CAPEX build-up is `$14,400 GPUs + $2,500 thermal BOM + $1,100 install = $18,000`. The host compute parts (CPU, motherboard, 384 GB ECC RAM, NVMe, redundant PSU, NIC, cellular modem, TPM, BMC) for an 8-GPU server realistically run **~$3,500–4,500** at this scale — which the $2,500 "thermal BOM" line as written does **not** cover. This is a real gap in the memo's BOM:

> **⚠ BOM reconciliation flag.** Either (a) the commercial CAPEX is higher than $18K once a genuine 8-GPU host is priced (true installed cost likely **~$20–22K**), or (b) the $2,500 "thermal BOM" line must be re-scoped to explicitly include the compute host and the real number found. Recommended: split the memo's single $2,500 line into **"compute host (~$3,500–4,500)" + "thermal balance-of-system (~$2,500)"**, restate commercial CAPEX to ~$20–22K, and re-run payback/IRR/DSCR. This is the single most important number to firm up before financing, because the tape's advance rate is struck against it. Tracked in §9 and `financial-and-roadmap.md`.

The $18K figure is preserved here for continuity with the memo and Slide 8, with the flag above so underwriting is not surprised.

---

## 6. Power, electrical & networking hardware

### 6.1 Power & electrical

- **Per-GPU draw — nominal vs design point:** **~412 W nominal** (economics memo; the basis for the ~3.3 kW envelope and the tape's energy accounting) and **~450 W design point** (pricing research peak/transient). **Size PSU and breakers to the 450 W design point; account energy/heat at the metered wall draw** (between the two). This is the single resolution of the 3.3 kW-vs-4 kW figures used throughout this doc (§1, §4).
- **H1:** the 4090 + host (~600 W total at the design point) sits inside a **3.3 kW heating system** already on a **240 V / 30 A water-heater branch circuit**. The GPU load is a small fraction of the heating element's draw, so no new service is needed — the unit fits the existing water-heater circuit. The heating element and the GPU are on **separate control domains** (principle 2): the element can heat with the compute node dead.
- **Commercial:** at the design point 8 × 450 W ≈ **3.6 kW GPU** + ~0.4 kW host ≈ **~4 kW node** (vs ~3.3 kW GPU at the 412 W nominal figure). On 240 V single-phase ~4 kW is ~17 A continuous, so the **base configuration is one 240 V / 40 A circuit with margin** (a single-PSU node, or both PSUs on the one circuit); larger sites use **208 V 3-phase**.
- **PSU redundancy and circuits.** The single-circuit case above is the *base* config. **Commercial-premium sites that want true PSU redundancy run the two redundant PSUs from two separate circuits** (each circuit sized to carry the full ~4 kW node alone), so a single branch trip doesn't drop the node. This is a premium-tier option, not the base build — there is no contradiction: base = one circuit, premium-redundant = two.
- **Inrush & transients.** 4090s exhibit large transient spikes; PSU and breaker headroom (1000 W H1, 2× redundant commercial) is sized for this, not just average draw. Soft-start / staggered GPU power-up is a thermal-controller responsibility (§7) to avoid tripping the branch breaker when 8 cards spin up at once.
- **Brownout / power-quality.** Residential power is dirtier than a data center's. The node tolerates sags by **failing compute first** (drop to heater-only), never the heater. No UPS is specced for H1 (the heater doesn't need one); commercial premium sites may add a small UPS sized only for graceful checkpoint-and-shutdown of GPU jobs (so interruptible work checkpoints cleanly on an outage), not for ride-through.

### 6.2 Networking hardware

Per principle 1, all connectivity is **outbound-only** — detailed in `connectivity-and-data.md`. Hardware requirements here:

- **H1:** onboard **2.5 GbE** to the home router (Wi-Fi as a fallback only; wired strongly preferred for upload stability). The link is whatever the homeowner has; the control plane *measures and scores* it rather than requiring a minimum. Upload is the bottleneck — see overview §9.
- **Commercial:** **primary wired link + mandatory cellular (4G/5G LTE) failover** via an auto-failover router. Rationale: a commercial node is (or aspires to be) a *high-reliability/premium* node carrying contracted, non-interruptible work; a single residential/business link is a single point of failure that would drop its reliability tier and its $/GPU-hr. The cellular link is a low-bandwidth control/keepalive path that keeps the node *reachable and meterable* during a primary-link outage (so the tape doesn't show a false outage and jobs can drain gracefully), not a path for bulk data egress. Cellular data is metered and budgeted as opex (`connectivity-and-data.md`).
- **No inbound hardware.** No public IP, no port-forwarding, no inbound firewall config required on the customer's router — the node dials out to the WireGuard overlay. This is a hard requirement for installability (the installer must not have to touch the customer's router config).

---

## 7. Thermal-coupling interface (sense & control, not plumbing)

The mechanical heat path (waterblock/coldplate → loop → tank) is out of scope. What the **hardware must provide** is the sensing and actuation that lets the node agent (Part B) and `platform.md`'s thermal coordinator treat the tank as a battery (principle 3, overview §8). The agent-side message contract to this board is in §16.

### 7.1 What the node must sense

| Signal | Sensor | Use |
|---|---|---|
| **Tank temperature** (1–3 zones for stratification) | Immersion/strap thermistors → thermal MCU | Storage headroom: how much GPU heat can the tank still absorb? Drives "can I run compute now / for how long." |
| **Heat demand / draw** | Flow sensor + inlet/outlet temp, or call-for-heat signal | Real-time hot-water/space-heat demand → thermal overlap accounting (the 70% number). |
| **GPU power & temp** | GPU telemetry (NVML) + wall-power meter | Power into the loop (= heat out), GPU junction temp, util → telemetry + safety. |
| **Coolant/loop temp & flow** | Loop thermistor + pump tach | Detect loop fault (pump failure, air, blockage) → trip compute before the GPU cooks. |
| **Ambient / enclosure** | Ambient thermistor | Climate-variant logic (§7.5 region variants); detect heat dumping to room. |

### 7.2 What the node must control

- **GPU power state:** the thermal controller can throttle (power-limit via NVML) or fully stop GPU compute, per-GPU on commercial. This is the actuator the thermal coordinator and yield router use to fit compute into thermal windows.
- **Heat-dump fallback:** if compute is wanted but the tank is full (no thermal headroom) and no heat is being drawn, the node must be able to either (a) shed the job, or (b) dump heat via the conventional path if the mechanical design allows. Policy lives in software; the hardware must expose the control line.
- **Staggered GPU power-up** (commercial) to manage electrical inrush (§6.1).

### 7.3 Safety interlock — local hardware wins

A dedicated **hardware safety interlock** in the thermal MCU enforces hard limits independent of the host OS and the cloud:

- Over-temp (GPU, loop, or tank) → cut GPU power.
- Pump failure / no-flow with GPUs hot → cut GPU power.
- Loss of agent heartbeat for N seconds with GPUs running → cut GPU power.
- **Default-safe:** on any thermal-MCU fault, GPUs are de-powered and the unit reverts to ordinary water-heater operation. **Local safety always overrides remote scheduling** (overview §4.2, principle 2). The recovery ladder (watchdog → A/B rollback → remote console → heater-only fallback, §16) bottoms out here in hardware.

### 7.4 Telemetry as a BOM requirement

Per principle 4 and the securitization tape: the node ships with the sensors above plus a **wall-power meter** so it can self-report, per the metering ledger in `platform.md`: GPU-hours delivered, power in/out, tank temp, heat draw, util, uptime, and link quality. Accurate, per-unit metering from day one is a *hardware* requirement because it is the collateral record. The software producer of these records is the node agent (§15).

### 7.5 Per-region / climate hardware variants

Standardize the SKU, vary a small, enumerable set of climate options (tracked in the registry per node, `platform.md`):

| Variant axis | Cold climate | Warm climate | Why |
|---|---|---|---|
| Thermal load | DHW + hydronic space heating tie-in (bigger absorber/buffer) | DHW-only (smaller heat sink) | Cold homes absorb ~12,000 kWh/yr (memo); warm/DHW-only ~4,500 kWh/yr → far less thermal overlap, so warm-climate nodes lean spot/interruptible. |
| Summer behavior | space-heat off in summer → DHW-only overlap | always DHW-only | Summer is the worst overlap window; the utilization engine (`platform.md`) biases these hours to VPP/demand-response or paid-electricity rented hours (overview §8). |
| Networking | as base | as base | — |
| Enclosure | as base | extra ambient-shed margin if room temps high | Avoid room overheating where space heat isn't wanted. |
| Cellular failover | commercial: yes | commercial: yes | Unchanged. |

The variant is a **configuration of the same SKU** (sensor population, absorber sizing, software profile), *not* a different board/host — preserving the homogeneous-collateral requirement. Heat-demand seasonality is the per-region modeling input the overview (§12) and `financial-and-roadmap.md` flag; the hardware's job is to *measure* it accurately per node so the financier's utilization promise is honest.

### 7.6 Per-SKU thermal constant (this doc is the anchor)

The tank-as-battery math — both the node's local control loop (§14.3) and the downstream thermal coordinator (`platform.md`, the utilization engine / control plane) — must use **one** capacity number per SKU, sourced here and imported by the control plane — not hardcoded in each consumer. The canonical values live in `data-contracts.md` §6; this hardware section is the physical anchor that supplies them:

- **`usable_thermal_kwh` ≈ 8.8 kWh** for the standard **80 US gal** H1 tank (C_th ≈ 0.352 kWh/°C × ΔT_usable ≈ 25 °C, e.g. 40→65 °C). This is the per-SKU value the control-plane registry stores (`data-contracts.md` §6 — the prior hardcoded `12.0` is retired). Larger/commercial-tank SKUs carry their own measured value.
- **`η_capture` ≈ 0.95** is a **placeholder** for the fraction of GPU heat actually captured into the loop; it **must be measured in Phase A** (flagged here as this doc owns the measurement). Every duty-cycle / overlap conclusion that uses it is conditional until measured.

The node reports `thermal.storage_headroom_kwh` derived from `E_store` (`data-contracts.md` §5, §6); the hardware's job is to populate the constants honestly per SKU and per region variant (§7.5), and the agent's job is to compute and report headroom from them (§14.3, §15).

---

## 8. Year-5 modular GPU swap

The financing model funds a **GPU-refresh reserve** for a year-5–6 module swap that re-ups the revenue curve for a second cycle (memo §2.4, §3.5). The hardware must make that swap a cheap, low-skill field operation — because thousands of swaps across scattered sites is otherwise an opex disaster.

### 8.1 Design-for-swap requirements

- **GPU as a field-replaceable module.** Cards mount in a **tool-light, quick-release carrier** (slide-in coldplate/waterblock mate, blind-mate or short-run power, captive thumbscrews) so a certified installer swaps a card in minutes without re-plumbing the loop. The mechanical mate to the heat path is a **dry-disconnect or quick-coupler** so the coolant loop need not be drained for a card swap (mechanical detail owned elsewhere; the *requirement* is "swap a GPU without breaking the loop").
- **Electrical & PCIe headroom for the successor card.** The H1's 1000 W PSU and the commercial redundant PSUs are sized so a **~550 W 5090-class successor drops in with no PSU or circuit change**, and PCIe slot/lane budget targets that successor. The heater shell's **~3.3 kW thermal envelope** is the real *thermal* ceiling — chosen deliberately so the heat path already accommodates more powerful future cards. **A ~1 kW Blackwell-class card (the pitch's 1 GW assumption) is NOT a drop-in:** it exceeds the H1's 1000 W PSU and may exceed the 240 V / 30 A branch, so it needs a PSU and possibly a circuit upgrade. The thermal path has room; the *electrical* path does not. See §9.7 — whether the H1 should carry PSU/circuit headroom *now* to be 1 kW-swap-ready is an open question.
- **Swap doesn't touch the safety domain.** A GPU swap must not require re-certifying the thermal interlock or the heater control — the two domains stay separable (principle 2).
- **No SKU lock-in to one card.** The carrier and host target *PCIe consumer cards generally*, not a single board revision, so the year-5 card can be whatever is cheapest-per-hour then (memo: falling card prices also cut the swap CAPEX 40%+).

### 8.2 Version-aware re-registration

A swapped card is a **new asset in a new generation** and the system must know it. This is a hardware-makes-it-swap-detectable + software-does-the-handshake operation; the software side is detailed in §17. On swap:

1. The agent detects the new GPU (PCI ID, VBIOS, VRAM, NVML capabilities) and reports it to `platform.md`'s registry (the control plane).
2. The control plane **re-registers the node at its new generation/SKU**, tags it to the relevant financing tranche / refresh cycle, and updates the metering tape's collateral line (old cohort retired, new cohort begins).
3. Reliability scoring resets the *seasoning* clock appropriately (a refreshed node carries its site/network history but a fresh card-reliability baseline).
4. The yield router and scheduler immediately reprice the node at the new card's tier (e.g., a 4090→5090 swap lifts it from $0.40 to $0.53-class demand).

This is overview §12's "version-aware re-registration in the control plane" requirement, met by hardware (the swap-detectable card) + software (the registry handshake, §17). It is also a **securitization covenant**: the refresh reserve pays for the swap, and the tape must show the collateral generation change cleanly. See `platform.md` for the registry handshake and `financial-and-roadmap.md` for the refresh-execution risk.

---

## 9. Open questions (hardware)

1. **Commercial CAPEX truth (the big one).** A real 8-GPU host (CPU, mobo, 384 GB ECC, NVMe array, redundant PSU, cellular, BMC) is ~$3,500–4,500 — not covered by the memo's $2,500 thermal-BOM line. **Is true commercial CAPEX ~$20–22K, not $18K?** Resolve before financing; the advance rate is struck against this number (§5.2 flag).
2. **8 vs 12 GPU — final freeze.** Recommendation is 8 (§4). The business must ratify before hardware freeze; if 12, the entire economics memo and Slide 8 must be re-underwritten and the 12-GPU XL given its own BOM/thermal/3-phase study.
3. **Refurb 4090 supply at scale.** Can a sustainable refurb channel deliver 10,000+ cards at ~$1,800 with acceptable failure rates and warranty? If not, the launch SKU may need to shift to 5090 (~$2,400) earlier, changing CAPEX and the GPU count per kW.
4. **Host parts as field-replaceable too?** The design centers GPU swap. Should CPU/RAM/NVMe/PSU also be field-serviceable, or is a host failure a whole-node RMA? Trade-off: serviceability cost vs. truck-roll cost across scattered sites.
5. **Cellular failover bandwidth/cost envelope.** What's the per-node cellular opex, and is "control/keepalive only" enough, or do premium contracted jobs need cellular bulk-egress fallback (much costlier)? Interacts with `connectivity-and-data.md` relay budget.
6. **Coolant quick-disconnect reliability over a 10-year, 2-swap life.** Dry-disconnect couplers must survive years and at least one swap cycle without leaks near electronics. Mechanical, but it gates the swap design (§8).
7. **PSU sizing for the year-5 card.** If the successor is a ~1 kW Blackwell-class card (pitch's 1 GW assumption), the H1's 1000 W PSU and the 240 V/30 A branch may *not* suffice for a 1-GPU home unit. Does the H1 need a PSU/circuit headroom margin now to be swap-ready then, or is the home unit capped at 5090-class?
8. **BMC vs agent recovery on commercial.** Does the commercial host need a true BMC/IPMI for out-of-band recovery, given the OS already does A/B rollback + remote console over the overlay (Part B, §16)? BMC adds cost but adds a recovery rung independent of the host link.
9. **TEE / confidential compute.** 4090s lack a robust hardware TEE (overview §10). Hardware can't fix this; the question is whether a future "trusted-host" commercial tier uses TEE-capable cards — a different (non-4090) BOM tracked in `financial-and-roadmap.md`.

---

# Part B — Software & OS

This part inherits the same five principles (§0) and treats them as acceptance criteria for every component below. It expands §4 of `system-design-overview.md`. The governing constraint repeats: **a node may be physically unreachable for months; every mechanism here recovers without a truck roll.**

## 11. Operating system

### 11.1 Why immutable / atomic

A hand-administered node is a dead node the first time an `apt upgrade` half-applies over a flaky residential uplink and the box won't reboot. We need an OS where:

- The root filesystem is **read-only and verified** (dm-verity or equivalent), so a corrupt write can't silently rot the system.
- Updates are **atomic** — they either fully apply to an offline partition or they don't apply at all; there is no half-state.
- There is **no general-purpose package manager** on the running node. The deployable unit is a whole signed image, built and tested in CI, not a set of packages assembled in the field.
- The OS can **roll back by itself** on a bad boot.

### 11.2 Candidate comparison

| | **Ubuntu Core** | **Flatcar Container Linux** | **Talos-style appliance** |
|---|---|---|---|
| Model | Snap-based, transactional | Image-based A/B (CoreOS lineage) | API-only, no shell/SSH at all |
| Root FS | Read-only, snap-confined | Read-only, dm-verity | Read-only, ephemeral |
| Updates | Per-snap + kernel, auto-rollback | Whole-image A/B, auto-rollback | Whole-image A/B, declarative |
| Mgmt surface | snapd + limited | Ignition (first boot) + systemd | gRPC API + machine config only |
| NVIDIA driver | Snap or baked layer | Baked into image (sysext/overlay) | Baked extension |
| Shell access | Yes (can disable) | Yes | **None** (forces good ops hygiene) |
| Fleet fit | IoT-oriented, good | Server-oriented, mature A/B | K8s-oriented; opinionated |
| Risk for us | snap store dependency, per-snap rollback granularity is fiddly | well-trodden A/B, GPU layering is manual but well-documented | great isolation but assumes a K8s control plane we don't run on-node |

### 11.3 Recommendation: **Flatcar-style image-based A/B**, customized

We recommend a **Flatcar-derived immutable image with whole-image A/B updates**, for these reasons:

- **Whole-image A/B is exactly the rollback primitive we need** and is the most battle-tested of the three for "appliance that updates itself in the field." Per-snap rollback (Ubuntu Core) has finer granularity than we want — we'd rather reason about *one* image hash per node, which also makes attestation (§18) and the securitization tape (`platform.md`) trivially clean: a node's identity includes exactly one running image digest.
- **We do not run Kubernetes on the node.** A single 4090 host with one tenant container at a time does not justify a K8s control plane, so Talos's core assumption doesn't fit. We borrow Talos's *philosophy* (no SSH in steady state, API-driven, ephemeral) but not its runtime.
- **GPU driver layering is a solved, documented problem** on Flatcar (sysext / overlay or baked layer), and we *want* the driver baked and version-pinned per image anyway (§17).

We adopt one Talos idea explicitly: **steady-state has no interactive shell.** All node interaction goes through the node agent's local API or, in recovery only, a console gated behind the reverse tunnel (§16, `connectivity-and-data.md`). This shrinks attack surface and removes "someone SSH'd in and changed something" as a class of fleet drift.

> Decision is reversible at the image-build layer: the agent, telemetry schema, and control-plane contracts below are OS-agnostic. If we later prefer Ubuntu Core, only the image-build pipeline and the A/B driver in §12 change.

### 11.4 Partition layout

```
┌──────────────┬───────────────────────────────────────────────┐
│ ESP (EFI)    │ bootloader + boot env (A/B pointer, try-count) │
├──────────────┼───────────────────────────────────────────────┤
│ USR-A (ro)   │ OS image slot A  (dm-verity, signed)           │
│ USR-B (ro)   │ OS image slot B  (dm-verity, signed)           │
├──────────────┼───────────────────────────────────────────────┤
│ OEM          │ node identity, provisioning data (read-mostly) │
├──────────────┼───────────────────────────────────────────────┤
│ STATE (rw)   │ agent state DB, logs, config overlay           │
├──────────────┼───────────────────────────────────────────────┤
│ CACHE (rw)   │ model/dataset/image cache (connectivity-and-data.md), purgeable│
├──────────────┼───────────────────────────────────────────────┤
│ SCRATCH      │ tenant NVMe scratch, wiped per job             │
└──────────────┴───────────────────────────────────────────────┘
```

`STATE`, `CACHE`, and `SCRATCH` survive OS updates; the two USR slots are the only thing an OTA touches. **A water-heater-safe minimal init lives in *both* USR slots and in a fallback path** so that even a double image failure still boots far enough to put the unit into heater-only mode (§13, §16).

---

## 12. A/B update + automatic rollback

### 12.1 Boot flow with health gate

The core invariant: **a new image is only made permanent (`active`) after it proves itself healthy on real boot.** Until then it is `active=tentative` with a boot try-counter.

```
        ┌─────────────────────────────────────────────────────────┐
        │ OTA: agent downloads image N to the INACTIVE slot,        │
        │ verifies signature + dm-verity hash, sets that slot       │
        │ active=tentative, boot_attempts=0, then reboots           │
        └───────────────────────────┬─────────────────────────────┘
                                     ▼
                      bootloader: which slot is active?
                                     │
                    ┌────────────────┴─────────────────┐
                    │ tentative & attempts < MAX(=3)    │
                    │ → boot it, attempts++             │
                    └────────────────┬─────────────────┘
                                     ▼
                       agent starts → runs HEALTH GATE
                                     │
              ┌──────────────────────┴──────────────────────┐
              ▼                                              ▼
         GATE PASS                                      GATE FAIL
   agent: bootloader_commit()                  agent does NOT commit;
   (active=permanent, attempts=0)              triggers reboot
   reports OTA success to ctrl plane                  │
                                                       ▼
                                         attempts hits MAX, or watchdog
                                         fires (agent never committed)
                                                       │
                                                       ▼
                                   bootloader: AUTO-ROLLBACK to the
                                   previously-permanent slot (N-1),
                                   marks slot N as bad
                                                       │
                                                       ▼
                                   agent on N-1 reports "rollback from N,
                                   reason=<...>" to control plane (§16)
```

The two independent rollback triggers are:

1. **Hard rollback (bootloader):** the agent fails to call `bootloader_commit()` within `MAX` boot attempts — covers the case where the new image won't boot or hangs before the agent runs. This requires *no working agent*, which is the point.
2. **Soft rollback (agent):** the agent boots, runs the health gate, fails a **heater-critical** check, and reboots *without* committing — covers "boots fine but the heater link / overlay / watchdog is broken." After enough non-commits the hard trigger takes over. A **compute-optional** failure does *not* roll back: the agent commits the image and boots into heater-only/compute-disabled mode (§12.2), so a dead GPU never costs the customer hot water or burns the boot-attempt budget.

### 12.2 Health gate

The health gate is a declarative, image-bundled checklist the agent runs **before** committing a new image. It is intentionally conservative: anything ambiguous fails the gate.

The gate splits its checks into two **classes**, because principle 2 (never lose hot water) outranks "deliver compute": a failed *heater-critical* check means the image is unsafe and must roll back; a failed *compute-optional* check means the GPU stack is broken but the unit can still heat, so the right answer is to **commit and boot into heater-only / compute-disabled mode and alert** — never to roll back into a boot loop over a dead GPU.

```yaml
# /etc/superheat/health-gate.yaml  (shipped inside the image, signed)
version: 4
timeout_s: 180

# HEATER-CRITICAL: any failure here ⇒ the image is unsafe ⇒ do NOT commit ⇒ roll back.
heater_critical:
  - id: agent_self
    desc: "agent process up, local API responding"
    cmd: "agentctl ping"
    must: exit0

  - id: thermal_iface
    desc: "thermal controller link is alive (see §16) — without it we are blind/unsafe"
    cmd: "agentctl thermal status --json"
    expect_json: { link: "ok", heater_safe: true }

  - id: watchdog_armed
    desc: "hardware watchdog is pettable"
    cmd: "agentctl watchdog selftest"
    must: exit0

  - id: overlay_up
    desc: "outbound overlay dials out & reaches control plane (else we can't be managed/recovered)"
    cmd: "agentctl net check --control-plane"   # connectivity-and-data.md
    must: exit0
    retries: 3

# COMPUTE-OPTIONAL: failure here ⇒ GPU stack broken but heating is safe ⇒
# COMMIT into heater-only/compute-disabled mode + alert. Do NOT roll back.
compute_optional:
  - id: nvidia_driver
    desc: "driver loads and matches pinned version"
    cmd: "nvidia-smi --query-gpu=driver_version --format=csv,noheader"
    expect_regex: "^550\\.(9[0-9]|1[0-9]{2})\\."

  - id: cuda_runtime
    desc: "CUDA runtime + tiny kernel launches"
    cmd: "superheat-cuda-smoketest"     # ~2s vector-add on device 0
    must: exit0

  - id: gpu_visible
    desc: "exactly the expected GPU count present"
    cmd: "nvidia-smi --query-gpu=count --format=csv,noheader"
    expect_equals_node_inventory: true   # 1 for H1, N for commercial

policy:
  # corrected gate logic — note rollback fires ONLY on heater-critical failure
  on_heater_critical_fail:
    action: rollback          # do NOT commit; reboot toward auto-rollback (§12.1)
    report: true              # best-effort report over overlay before reboot
  on_compute_optional_fail:
    action: commit_degraded   # COMMIT the image, but enter HEATER-ONLY SAFE-MODE (§14.4)
    compute_enabled: false    # node will not be assigned paid work until GPU stack recovers
    alert: true               # raise a fleet alert + flag node for inspection (rung 3, §16)
    report: true
  on_all_pass:
    action: commit            # commit, compute enabled
```

So an image that can heat-safe and phone home but has a broken GPU **commits and degrades to heater-only/compute-disabled** (and alerts) rather than rolling back into a loop — the GPU being dead is not a reason to throw away an otherwise-good, safe image. See §14.4 transition into `HEATER-ONLY SAFE-MODE`. Only a *heater-critical* failure triggers the rollback path of §12.1.

### 12.3 Staged rollout coupling

The node-side mechanism above is the enforcement arm of the control plane's staged rollout (canary → cohort → fleet, `platform.md`). The node:

- only ever pulls the image digest the control plane has assigned to its cohort;
- reports `ota_result` with `{from_digest, to_digest, gate_results[], committed|rolled_back|committed_degraded, duration_s}`;
- a fleet-wide spike in `rolled_back` is the control plane's automatic halt signal.

### 12.4 Post-commit health-regression watchdog

The gate (§12.2) and boot try-counter (§12.1) only catch failures that show up **at boot**. An image can pass the gate and then go unstable *hours* later — a thermal-link flap, a creeping XID-error rate, repeated agent crashes, or a slow memory leak. Two mechanisms cover this:

1. **Health-regression watchdog (node-side, auto-revert after commit).** For a **soak window** after a commit (e.g., 24 h, tunable), the agent keeps a pointer to the previously-permanent slot (N−1) and runs a continuous health monitor. If it crosses a regression threshold within the window — e.g., ≥ N agent crash-loops, sustained `xid_errors` above baseline, repeated thermal-link loss, or the compute-optional checks (§12.2) start failing on a previously-healthy GPU — the agent treats it as a **late gate failure**: it drains, marks slot N bad, and triggers a rollback to N−1 via the same bootloader path as §12.1, reporting `ota_result.rolled_back` with `reason=post_commit_regression`. Crucially this **only ever reverts the image**; it never disables heating — a regression that is heater-critical drops to heater-only safe-mode (§14.4) just like any other safety trip. After the soak window expires cleanly, N−1 is released and the image is fully trusted.

2. **Staged cohort rollout (fleet-side, catches what the soak window misses).** Failures slower than the per-node soak window (days-scale leaks, load-correlated faults) are caught by the control plane's canary → cohort → fleet rollout (`platform.md`): a rise in `rolled_back` / `committed_degraded` / fault-rate in an early cohort halts promotion before the image reaches the fleet. The honest limitation: a regression that is *both* slower than the soak window *and* uncorrelated across the cohort can still slip through to inspection (rung 3, §16) — staged rollout narrows but does not fully close that gap.

---

## 13. The node agent

A **single signed daemon** (`superheat-agent`) owns the node. It is the only privileged long-running process; tenants never run as it and never see its socket.

### 13.1 Responsibilities (overview §4.2, made concrete)

- **Heartbeat & telemetry** — periodic signed report of power, GPU temp/util, tank temp & heat demand, net quality (the canonical `superheat.telemetry.v1` frame, `data-contracts.md` §5; §15 here), plus the signed `superheat.usage.v1` billing record (`data-contracts.md` §8).
- **OTA orchestration** — §12: pull, verify, A/B apply, gate, commit/rollback, report.
- **Container/microVM lifecycle** — launch/stop/preempt tenant workloads with isolation (§14 — see §14.5/§14.6/§14.7).
- **Thermal & power policy enforcement** — the local control loop (§14.3): obey the thermal coordinator's plan *subject to* hard local limits.
- **Local job cache** — manage `CACHE` partition for base images, weights, datasets (`connectivity-and-data.md`).
- **Hardware watchdog** — pet a hardware watchdog; if the agent dies or hangs, the box auto-reboots.
- **Identity & attestation** — hold node keys, answer attestation challenges before paid work (§18, `security-and-compliance.md`).

### 13.2 Internal architecture (modules)

```
                         ┌───────────────────────────────────────┐
                         │            superheat-agent             │
                         │                                        │
  overlay (connectivity) │  ┌─────────────┐   ┌────────────────┐  │
  ◄──────────────────────┼─►│ ControlLink │   │   Supervisor   │  │  ← owns the
   control / OTA / console│  │ (mTLS, gRPC │   │ (state machine │  │    state machine
                         │  │  outbound)  │   │   §14.4)       │  │
                         │  └──────┬──────┘   └───┬────────┬───┘  │
                         │         │              │        │      │
                         │  ┌──────▼──────┐  ┌────▼───┐ ┌──▼────┐ │
                         │  │ Telemetry   │  │ Job-   │ │ OTA   │ │
                         │  │ Collector   │  │ Runtime│ │ Mgr   │ │
                         │  └──────┬──────┘  └───┬────┘ └───┬───┘ │
                         │         │             │          │     │
                         │  ┌──────▼─────────────▼──────────▼───┐ │
                         │  │     ThermalPowerController         │ │  ← LOCAL SAFETY
                         │  │  (the arbitration loop, §14.3)     │ │    AUTHORITY
                         │  └──────┬───────────────────┬────────┘ │
                         │         │                   │          │
                         │  ┌──────▼──────┐     ┌───────▼───────┐  │
                         │  │ HeaterIface │     │ WatchdogMgr   │  │
                         │  │ (§16 board) │     │ (HW + SW dog) │  │
                         │  └─────────────┘     └───────────────┘  │
                         │  ┌────────────────────────────────────┐ │
                         │  │ StateStore (embedded, on STATE part)│ │
                         │  └────────────────────────────────────┘ │
                         └────────────────────────────────────────┘
                              │                         │
                       ┌──────▼──────┐           ┌──────▼──────┐
                       │ NVIDIA      │           │ Heater      │
                       │ container/  │           │ control     │
                       │ microVM     │           │ board (MCU) │
                       └─────────────┘           └─────────────┘
```

- **ControlLink** — the only network egress for control: outbound mTLS/gRPC over the WireGuard overlay (`connectivity-and-data.md`). Carries heartbeat, command stream, OTA negotiation, console multiplexing. Reconnect-with-backoff; survives IP changes/CGNAT.
- **Supervisor** — runs the state machine (§14.4); the single decision point for what the node is doing.
- **TelemetryCollector** — samples GPU (NVML), host, power meter, and HeaterIface; emits reports (§15).
- **JobRuntime** — talks to the container/microVM runtime (§14.5–§14.6); enforces resource caps and egress policy; handles checkpoint/preempt (`connectivity-and-data.md`). On preempt it gives an interruptible job the canonical graceful-stop budget **`T_GRACE_S = 30`** (`data-contracts.md` §7) to flush its in-flight checkpoint; because 30 s cannot finish a multi-minute upload, durable resume relies on the last PoP-persisted checkpoint, not the in-flight flush (`connectivity-and-data.md`).
- **OTAMgr** — §12.
- **ThermalPowerController** — the local control loop (§14.3). **It can veto JobRuntime.** This is where principle 3 + the "local safety wins" rule physically live.
- **HeaterIface** — the message bus to the heater control MCU (§16); the hardware board behind it is §7.
- **WatchdogMgr** — pets the hardware watchdog and runs internal liveness checks (§16).
- **StateStore** — small embedded DB (e.g., SQLite/BoltDB) on the `STATE` partition; persists state, last-known-good config, pending OTA, job ledger fragments.

---

## 14. The local control loop, arbitration, isolation & state machine

### 14.1 Overview

This section is the heart of the agent: the arbitration loop that turns sensor readings, heater status, and the remote plan into a single GPU power action (§14.3); the state machine that governs what the node is doing (§14.4); and the tenant-isolation runtime the loop schedules work into (§14.5–§14.7).

### 14.3 The local control loop (arbitration)

This is the literal implementation of *"local safety ALWAYS wins over remote scheduling."* It runs on a fast fixed tick (e.g., 1 Hz) and is the **only** path that can change GPU power state.

```
every tick:
  s   = read_sensors()         # GPU temp, board power, tank temp, heat demand, ambient
  hw  = HeaterIface.status()   # heater_safe flag, tank setpoint, comfort state (§16)
  plan = Supervisor.current_assignment()   # what remote scheduler wants (may be none)

  # ---- HARD SAFETY HALTS (inviolable, evaluated FIRST; always outrank THROTTLE) ----
  if   s.gpu_temp  > GPU_TEMP_CRIT      : action = HALT_GPU   # over-temp
  elif s.tank_temp > TANK_TEMP_MAX      : action = HALT_GPU   # can't dump more heat safely
  elif hw.heater_safe == false          : action = HALT_GPU   # MCU demands heater priority
  elif not hw.link_ok                   : action = HALT_GPU   # blind = unsafe → stop compute
  elif not hw.flow_ok                   : action = HALT_GPU   # no coolant flow → stop compute
  elif ControlLink.heartbeat_lost()     : action = HALT_GPU   # lost remote heartbeat → fail safe

  # ---- POWER THROTTLE (a clamp, never a halt; only after all hard halts pass) ----
  elif s.board_power > CIRCUIT_BUDGET_W + THROTTLE_HYST_W : action = THROTTLE  # near breaker → shed
  elif throttling and s.board_power > CIRCUIT_BUDGET_W - THROTTLE_HYST_W : action = THROTTLE  # hold until clearly below (hysteresis)

  # ---- THERMAL COUPLING (principle 3) ----
  elif tank_has_storage_headroom(s, hw) == false and \
       no_real_time_heat_demand(hw)             : action = HALT_GPU  # nowhere for heat to go

  # ---- REMOTE PLAN (only if all above are satisfied) ----
  elif plan.run and within_power_budget(plan, s): action = RUN(plan, power_cap=derive_cap(s, hw))
  else                                          : action = IDLE

  apply(action)      # power-limit GPU (nvidia-smi -pl), pause/kill job, signal HeaterIface
  WatchdogMgr.pet()  # only if loop completed cleanly
  TelemetryCollector.note(action, s, hw)
```

The two helper predicates are "the tank is a battery" made literal, using the canonical thermal constants in `data-contracts.md` §6 (`E_store ≈ 8.8 kWh` for the standard 80-gal H1 tank; per-SKU `usable_thermal_kwh` in the registry; physical anchor in §7.6):

```
tank_has_storage_headroom(s, hw):
    # remaining absorbable heat, same definition as telemetry storage_headroom_kwh (data-contracts §6):
    #   E_store * (T_max - T_avg) / ΔT_usable, clamped ≥ 0
    headroom_kwh = max(0, hw.E_store * (hw.T_max - s.tank_temp_avg) / hw.dT_usable)
    return headroom_kwh > MIN_HEADROOM_KWH        # small floor so we don't start a job we must abort in seconds

derive_cap(s, hw):
    # set the GPU power cap so heat OUT ≈ heat the tank can absorb right now (W), within the HW envelope.
    headroom_kwh    = max(0, hw.E_store * (hw.T_max - s.tank_temp_avg) / hw.dT_usable)
    absorb_w_budget = hw.realtime_demand_w + headroom_kwh * 1000 / TANK_FILL_HORIZON_H  # demand + bank, spread over horizon
    target_w        = absorb_w_budget / ETA_CAPTURE                                      # η_capture (data-contracts §6)
    return clamp(target_w, GPU_PCAP_MIN_W, GPU_PCAP_MAX_W)   # 412–450 W envelope, Part A §6.1
```

Key properties:

- **Hard safety halts always outrank the power throttle.** Over-temp, tank-full, lost thermal link, no coolant flow, and lost remote heartbeat each force `HALT_GPU` *before* the throttle branch is ever considered — a near-breaker condition can only ever *clamp* power, never override a safety halt.
- The remote scheduler's plan is an **input**, never a command that bypasses limits. The scheduler can ask for more than the node will give; the node clamps.
- **`derive_cap()`** continuously sets the GPU power limit (412–450 W envelope, see Part A §6.1) so the GPU's heat output matches what the tank can absorb *right now* — the GPU is throttled, not just on/off. This is "the tank is a battery" expressed as a power cap.
- **Throttle hysteresis.** The throttle engages above `CIRCUIT_BUDGET_W + THROTTLE_HYST_W` and only releases below `CIRCUIT_BUDGET_W − THROTTLE_HYST_W`, so the cap doesn't oscillate around the breaker limit tick-to-tick.
- **`CIRCUIT_BUDGET_W` is SKU-derived, not a magic number.** It comes from the node's branch circuit (Part A §6.1), not a literal. On **H1** the GPU and host share the existing **240 V / 30 A water-heater branch** — a continuous-load budget of `240 × 30 × 0.8 ≈ 5760 W`, of which the conventional heating element's share is reserved, leaving the compute envelope the cap clamps to. A **commercial** node is provisioned per-rack (~4 kW per node typical). The registry carries the per-SKU value; the loop never hardcodes one.
- If the loop itself stalls, the watchdog is **not** petted, and §16 takes over. A stuck control loop must never leave the GPU running blind.

### 14.4 Agent state machine

```
                       power-on / first boot
                              │
                              ▼
                    ┌───────────────────┐
                    │   PROVISIONING    │  get identity, attest, register (§18)
                    └─────────┬─────────┘
                              │ registered + attested OK
                              ▼
                    ┌───────────────────┐  ◄──── job finished / preempt done
            ┌──────►│       IDLE        │◄─────────────────────────┐
            │       └─────────┬─────────┘                          │
            │   assignment +  │ thermal window open                │
            │   thermal OK    ▼                                    │
            │       ┌───────────────────┐   drain request /        │
            │       │   RUNNING-JOB     │   thermal window closing /│
            │       └─────────┬─────────┘   higher-yield preempt    │
            │                 │                                     │
            │                 ▼                                     │
            │       ┌───────────────────┐  checkpoint to NVMe,      │
            │       │ DRAINING/PREEMPT  │  release tenant (data)   ─┘
            │       └─────────┬─────────┘
            │                 │ drained
            │                 ▼
            │       ┌───────────────────┐  A/B apply + health gate (§12)
            │       │     UPDATING      │
            │       └───┬───────────┬───┘
            │  gate pass│           │ gate fail / boot fail
            │  commit   │           ▼
            │           │   ┌───────────────────┐
            │           │   │    RECOVERING     │  watchdog→rollback→
            │           │   │  (recovery ladder │  console (§16)
            │           │   │       §16)        │
            │           │   └───┬───────────┬───┘
            │           │ recov.│           │ unrecoverable / unsafe
            │           │ OK    │           ▼
            │           ▼       │   ┌────────────────────────┐
            └───────────┴───────┴──►│  HEATER-ONLY SAFE-MODE  │
                                    │  GPU disabled; unit runs │
   ANY state, on hard safety trip ─►│  as a plain water heater │
   or loss of thermal link ────────►│  (principle 2 floor)     │
                                    └────────────┬─────────────┘
                                                 │ operator clears /
                                                 │ good image + safe sensors
                                                 ▼
                                              IDLE (re-attest first)
```

`HEATER-ONLY SAFE-MODE` is reachable from **every** state and is the absorbing safe state. In it: GPU power is hard-off, the JobRuntime is stopped, the HeaterIface runs the conventional heating element on its own MCU logic (it can do this even if the agent dies, see §16), and the agent keeps trying to phone home and self-heal. **The customer always has hot water in this state.**

### 14.5 Tenant isolation — baseline: containers via NVIDIA Container Toolkit

Default tenant runtime is OCI containers with the **NVIDIA Container Toolkit** (`nvidia-container-runtime`), matching `system-design-overview.md` §4.2:

- Tenants get the GPU device injected; they **never** install or touch the host driver — the driver/CUDA stack is baked and pinned in the host image (§17).
- Containers run rootless / with a locked-down profile: dropped capabilities, seccomp, no host PID/IPC/net namespaces, no access to the host filesystem beyond an explicit scratch mount, **no path to the HeaterIface socket or the agent's API**.
- Resource caps: cgroup CPU/RAM limits, GPU power cap is set by the control loop (§14.3) *outside* the container so the tenant cannot raise it.

### 14.6 Optional stronger isolation: microVMs

For the **trusted-host tier** and any multi-tenant slicing on commercial nodes (`security-and-compliance.md`), we support a microVM runtime — **Firecracker** or **Cloud Hypervisor** — fronted by Kata-style integration so the same OCI job spec runs in a VM with its own kernel.

Tradeoffs **specific to consumer GPUs**:

| | Containers (NVIDIA Toolkit) | microVM (Firecracker / Cloud Hypervisor) |
|---|---|---|
| Isolation strength | Shared host kernel | Separate guest kernel — much stronger |
| GPU passthrough | Native, trivial | Needs **VFIO PCIe passthrough**; the 4090 must bind to `vfio-pci` |
| Firecracker GPU support | n/a | **Firecracker has no PCIe passthrough** today → 4090 not usable |
| Cloud Hypervisor | n/a | **Supports VFIO** → can pass the whole 4090 to one guest |
| GPU sharing | MPS/MIG-style sharing possible | Whole-GPU only (no MIG on 4090); one tenant per card |
| Boot/overhead | ~instant | ~hundreds of ms + VFIO bind/unbind cost on preempt |
| Reset hygiene | driver-level | full VM teardown + GPU FLR between tenants |

**Recommendation:** on H1 (single-tenant, one 4090) the container path is sufficient and cheaper; reserve **Cloud Hypervisor + VFIO** for the trusted-host/commercial tier where stronger isolation justifies whole-GPU passthrough. **Firecracker is not viable for GPU tenants** (no passthrough) — note it only for non-GPU control workloads. This is consistent with the honest confidentiality limit in `system-design-overview.md` §10 and `security-and-compliance.md`: even a microVM does not give hardware-TEE confidentiality on a 4090; it raises the isolation bar, it does not make the host trustless.

### 14.7 Egress firewall

Every tenant's network is mediated by the agent (`security-and-compliance.md`, overview §10):

```
# conceptual per-tenant egress policy (enforced by agent, not tenant-editable)
default: DENY
allow:
  - to: regional-cache.superheat.net        # connectivity-and-data.md pre-staging / checkpoints
  - to: tenant-declared-allowlist[]          # renter-requested endpoints, rate-limited
deny-always:
  - home LAN (RFC1918 of the host's residential network)   # never touch the homeowner
  - the overlay control subnet / agent API
  - the heater MCU
  - known mining pools / abuse lists
log: all-connections   # for abuse enforcement + dispute evidence
```

The home LAN block is non-negotiable: a renter container must be unable to reach the homeowner's printer, NAS, or router. Egress goes out the overlay/relay path, not the homeowner's bare LAN, wherever possible (`connectivity-and-data.md`).

---

## 15. Telemetry & usage records (the agent is the producer)

The agent is the **producer** of two canonical wire objects defined NORMATIVELY in `data-contracts.md`; this doc does not redefine them, it implements them. The sensors that feed them are the Part A BOM requirement (§7.4).

### 15.1 Telemetry frame — `superheat.telemetry.v1`

The canonical telemetry frame is **`data-contracts.md` §5**: `gpus` (always an array, even on H1), per-GPU `mem_used_mb`, `thermal.mode` + `thermal.heat_demand_w` (mode enum *and* wattage), `board_power_w`, `net.path ∈ {direct,relay}`, `cloud_degraded`, and the `sig` over the node identity key (§18). The agent's `TelemetryCollector` (§13.2) populates this frame from NVML + HeaterIface (§16) + the power meter and emits it outbound-only on a cadence (e.g., 10–30 s heartbeat, richer report on state change). It is the source of truth for reliability scoring (`data-contracts.md` §3, `platform.md`).

```jsonc
// PROJECTION of data-contracts.md §5 — see there for the normative schema. Do not redefine fields here.
{
  "schema": "superheat.telemetry.v1",
  "node_id": "uuid",
  "agent_version": "1.0.0",
  "seq": 184221,                          // monotonic; gap = lost heartbeats
  "ts": "2026-06-18T14:03:21Z",
  "clock_skew_ms": 12,
  "gpus": [                               // ALWAYS an array (length 1 on H1)
    { "slot": 0, "model": "RTX4090", "util_pct": 97, "mem_used_mb": 19840,
      "power_w": 431, "power_cap_w": 450, "temp_c": 71, "xid_errors": 0 }
  ],
  "thermal": {                            // from HeaterIface §16
    "mode": "storing",                    // none|realtime|storing|full (enum)
    "heat_demand_w": 1200,                // instantaneous useful heat demand, WATTS
    "tank_temp_c": [58.4, 49.0],
    "storage_headroom_kwh": 2.1,          // = E_store*(T_max−T_avg)/ΔT_usable (data-contracts §6)
    "comfort_ok": true
  },
  "board_power_w": 612,                   // whole-node wall draw (circuit-budget enforcement §14.3)
  "net": { "path": "direct", "uplink_mbps": 11.2, "downlink_mbps": 184.0,
           "rtt_ms": 38, "loss_pct": 0.1 },
  "ckpt_age_s": 240,
  "cloud_degraded": false,                // set by aggregator on correlated outage
  "sig": "ed25519:..."                    // node identity key (§18)
}
```

The control plane treats missing `seq`/heartbeats, rising `xid_errors`, throttling, `net.path == "relay"`, and post-commit rollbacks (§12.4) as reliability-score inputs (`data-contracts.md` §3). Node-local fields not needed cross-boundary (running `image_digest`, agent `state`, `last_gate`) ride as agent-internal extensions, not part of the canonical frame.

### 15.2 Signed usage record — `superheat.usage.v1` (the billing primitive)

Telemetry is for **health/scoring**; **money** rides on the canonical signed **usage record** (`data-contracts.md` §8) — this is the metering primitive that feeds the securitization tape and was previously missing on the node side. The agent emits one `superheat.usage.v1` per (GPU slot, job, billing period), filling the metered facts (`gpu_slot`, `gpu_seconds`, metered `energy_kwh`, `gross/fee/net_usd`) and applying the **`node_sig`** with the node identity key. The control plane counter-signs (`cloud_sig`) after reconciliation, producing the double-signed provenance chain (`data-contracts.md` §8, `platform.md`). Per `data-contracts.md` §8 a record is **per-GPU** and **never spans a billing-period boundary** (split at the boundary); reconciliation tolerance applies to these signed records vs marketplace settlement, never to lossy telemetry.

---

## 16. Recovery ladder & thermal controller interface

### 16.1 Recovery ladder

Ordered from cheapest/most-automatic to last-resort, exactly as overview §4.3 — every rung self-acts with **no truck roll**:

```
 Rung 0  CONTROL LOOP CLAMP   (§14.3) bad sensor/over-temp/breaker → throttle/halt GPU, keep node up
            │ doesn't recover
 Rung 1  HARDWARE WATCHDOG     agent hung / kernel wedged → MCU/HW watchdog reboots the host
            │ reboots into bad image
 Rung 2  A/B AUTO-ROLLBACK     (§12) new image fails gate/boot → bootloader reverts to last-good
            │ rollback also unhealthy / needs eyes
 Rung 3  REMOTE CONSOLE        operator opens console over the REVERSE TUNNEL (connectivity-and-data.md):
            │                  outbound-initiated, no inbound port; inspect logs, push fix image
            │ unrecoverable in software / unsafe sensors
 Rung 4  HEATER-ONLY SAFE-MODE (§14.4) GPU disabled; unit runs as a plain water heater.
                               Customer keeps hot water; node triaged or scheduled for service.
```

Two hard rules:

- **The hardware watchdog and the heater MCU are independent of the agent.** If the entire agent/OS is gone, the watchdog still reboots and the MCU still heats water. Software can fail completely and the unit is still a working water heater (principle 2).
- **Rung 3 is outbound-only.** The console is a stream multiplexed over the same node-initiated overlay connection (`connectivity-and-data.md`); we never open an inbound port on the home router.

### 16.2 Thermal controller interface

The agent does **not** drive heating elements or pumps directly; that lives on a dedicated **heater control board (MCU)** that can run the unit as a conventional water heater entirely on its own. The agent and MCU exchange messages over a local link (e.g., UART/CAN/I²C; the hardware board and its sensing/control responsibilities are §7). The split exists so the heating function survives total agent/host failure.

#### 16.2.1 Message contract

Agent → MCU:

| Message | Payload | Meaning |
|---|---|---|
| `gpu_heat_intent` | `{watts, duration_hint_s}` | "I intend to dump ~W of GPU heat for ~T" — MCU plans element/pump accordingly |
| `request_window` | `{}` | "may I run compute now?" → MCU replies with headroom |
| `set_pump` | `{mode}` | request circulation for the GPU loop (advisory; MCU may override) |
| `agent_alive` | `{ts}` | liveness ping (also the agent's side of a cross-watchdog) |

MCU → Agent:

| Message | Payload | Meaning |
|---|---|---|
| `thermal_status` | `{tank_temp_c, setpoint_c, headroom_kwh, heat_demand}` | periodic state for the control loop & telemetry |
| `heater_safe` | `{bool, reason}` | **the master safety signal** — false ⇒ agent must HALT_GPU (§14.3) |
| `comfort_constraint` | `{min_tank_c, deadline}` | hard comfort floor the agent must respect |
| `fault` | `{code}` | sensor/element fault → agent moves toward heater-only safe-mode |

#### 16.2.2 Safety semantics

- `heater_safe=false` from the MCU is an **immediate, non-overridable** HALT_GPU in the control loop. The MCU is the local safety authority for heat; the agent is its compute client. This is the software counterpart of the hardware safety interlock in §7.3.
- **Loss of the link = unsafe.** If `agent_alive`/`thermal_status` stops flowing, the control loop treats `link_ok=false` → HALT_GPU, and the MCU's own watchdog (missing `agent_alive`) independently forces conventional-heating-only mode. Both sides fail safe toward "just a water heater."
- The MCU's firmware is updated through the same signed-image/OTA discipline but on a **separate cadence and slot**, so a host OTA can never leave the heater unable to heat.

---

## 17. Build & test pipeline (image-side)

Because the node has no package manager, **CI is the only way software reaches a node**, which makes the pipeline a first-class part of this design:

1. Build the immutable image: base Flatcar-derived OS + pinned **NVIDIA driver + CUDA runtime + container toolkit** + `superheat-agent` + signed `health-gate.yaml`.
2. Pin and record every version → produces a single **content-addressed image digest** (the unit of attestation §18 and rollout §12).
3. Test on a hardware-in-the-loop rig: run the **same health gate** the field will run, plus GPU smoke tests and a simulated thermal-link (see `operations.md` for the testing/simulation harness).
4. Sign the image; publish to the release set the control plane draws from.
5. Control plane rolls out canary → cohort → fleet; node-side A/B + gate (§12) is the safety net for anything CI missed.

Driver/CUDA upgrades are therefore *image* events, gated and rollback-protected like any other — never an in-place `apt`/`.run` install on a live node.

**GPU module swap (the software side of §8.2).** On a year-5 card swap, the agent detects the changed GPU (PCI ID, VBIOS, VRAM, NVML capabilities), re-attests (§18), and triggers version-aware re-registration with `platform.md`'s registry without a reinstall — the new generation, tranche tag, and reset seasoning clock land on the tape exactly as §8.2 describes.

---

## 18. Attestation, identity & secrets

(Full trust model in `security-and-compliance.md`; node-side mechanics here.)

> **Hardware dependency (be honest about it).** The attestation and sealed-secrets flow below **requires a hardware TPM / secure element** on the node — the BOM line in Part A §5 (it is listed for both SKUs). On any node *without* a hardware secure element, the key binding falls back to **software-only**, which is meaningfully weaker: the signing key lives in software-protected storage on the `OEM`/`STATE` partition, so a stolen disk + image is far harder to defend against and the "sealed-to-the-genuine-image" guarantee (§18.3) does not hold. Consequence: the fleet-wide *"clean by construction"* / hardware-attested property holds **only for the secure-element-equipped subset** of the fleet; software-only nodes are attested at a lower assurance level and the control plane must track which assurance class a node is in. See `security-and-compliance.md` for how the two classes map to confidentiality / trusted-host policy.

### 18.1 Identity provisioning

- Each node gets a **hardware-bound key** at manufacture/install: a key in a **TPM/secure element (required; see the dependency note above and Part A §5)**, with the public key registered to the node record in the control-plane registry (`platform.md`). Private key never leaves the node. On a node lacking the secure element, this is a software-held key and the binding is software-only/weaker (above).
- First boot enters `PROVISIONING` (§14.4): the agent uses an install-time enrollment token (one-time, short-lived) to establish its identity and obtain its long-lived overlay credentials and signing key material.
- Identity is the tuple `{node_id, hw_key_pub, sku, region, owner/tranche tag}` — the same record that anchors the securitization tape.

### 18.2 Attestation before paid work

A node must prove it's running genuine, signed Superheat software **before the scheduler hands it paying work** (overview §10):

```
control plane ──challenge(nonce)──►  agent
agent gathers measurements:
   - running OS image_digest (the committed A/B slot, §12)
   - agent binary measurement
   - secure-boot / dm-verity state
   - thermal-link present (heater_safe path intact)
agent ──signed_quote{nonce, measurements}── (hw key)──► control plane
control plane verifies:
   - image_digest ∈ allowed_release_set
   - signature chains to the registered hw_key
   - node not flagged / not in safe-mode
        │ pass                         │ fail
        ▼                              ▼
  mark ATTESTED, eligible        quarantine: no paid work,
  for paid scheduling            force re-image / inspect (rung 3)
```

Attestation is re-run on every image change (post-OTA commit) and periodically. A node that can't attest can still heat water; it just won't be assigned (or paid for) compute. This keeps the billing ledger honest: **only attested image+identity node-hours enter the tape.**

### 18.3 Secrets

- Long-lived secrets (overlay key, signing key) are **sealed to the TPM** and to the expected image measurement — they only unseal when the genuine image is booted, so a stolen disk yields nothing useful. **This guarantee depends on the hardware secure element (§18 dependency note);** on software-only nodes the secrets are software-encrypted at rest without the hardware unseal-gated-on-measurement property, so the "stolen disk yields nothing" claim is weaker there.
- **Per-job secrets** (e.g., a renter's pull token) are short-lived, scoped to the JobRuntime/container, injected via tmpfs, and wiped on job end together with `SCRATCH` (no-persistence-by-default, overview §10 / `connectivity-and-data.md`).
- Secrets are never written to the read-only image or to logs.

---

## 19. Open items (node-software-specific)

- **Commercial multi-GPU isolation:** whole-GPU passthrough per tenant (Cloud Hypervisor + VFIO) vs. shared-host containers with per-GPU assignment — depends on the 8-vs-12-GPU SKU decision (Part A §4, overview §3.3).
- **MCU↔agent bus choice** (UART/CAN/I²C) and its electrical isolation — Part A §7.
- **GPU module swap (year-5 refresh):** the agent must detect a changed GPU, re-attest, and trigger version-aware re-registration without a reinstall (Part A §8.2, §17, overview §12 refresh-reserve).
- **Watchdog timing constants** (`MAX` boot attempts, gate `timeout_s`, control-loop tick) need field tuning against real residential reboot/uplink behavior (see `operations.md`).
- **Power cap policy** (`derive_cap`, §14.3) — the predicate is defined against the canonical thermal constants (`data-contracts.md` §6), but its tunables (`TANK_FILL_HORIZON_H`, `MIN_HEADROOM_KWH`, `THROTTLE_HYST_W`) and per-SKU `CIRCUIT_BUDGET_W` need joint tuning with the thermal coordinator (`platform.md`) so banked-heat economics and circuit limits agree.
- **Post-commit soak window** (§12.4) — the regression thresholds and soak duration need field tuning against real residential stability, and must be reconciled with the fleet rollout cadence (`platform.md`).
- **Hardware secure element** (§18) — the TPM/secure-element BOM addition (now carried in Part A §5) and the assurance split between hardware-attested and software-only nodes is a hard dependency on `security-and-compliance.md`.

---

*Cross-references:* `system-design-overview.md` (§3 and §4 expanded here), `connectivity-and-data.md` (outbound-only link + cellular failover, the data/checkpoint path), `platform.md` (registry, scheduler/utilization engine, metering tape, version-aware re-registration, renter API), `security-and-compliance.md` (trust model, confidentiality classes, regulatory posture), `financial-and-roadmap.md` (refurb supply, CAPEX truth, 8-vs-12, refresh execution, opex/financial model), `operations.md` (observability/SLO/incident, testing/simulation, field operations/onboarding), `data-contracts.md` (normative telemetry/usage/thermal-constant schemas).
*Source consistency:* `../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md`, `../market-research/GPU_Pricing_Research_4090_H100.md`, `../market-research/Superheat AI Data Center.md`.
