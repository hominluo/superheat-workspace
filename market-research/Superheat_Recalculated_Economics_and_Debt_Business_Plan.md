# Superheat — Recalculated Unit Economics (Live GPU Market) + Detailed Debt & Securitization Business Plan

*v0.2 — internal working draft, June 2026.*
*Inputs: `Superheat intro_v3.5.pdf`, live Vast.ai pricing snapshot (saved 2026-06-11), and market research (sources at end).*
*Companion to: `Superheat_Compute-Network_Debt-Engine_Onepager.md`*

---

# Part I — What the live GPU market actually pays

## 1.1 Vast.ai live platform rates (snapshot, June 2026)

Median on-demand $/GPU-hr across 40+ data centers; ranges are min–max observed:

| GPU | VRAM | Median $/hr | Range | Availability |
|---|---|---:|---|---|
| B300 (Blackwell Ultra) | 288 GB | $6.52 | $5.33–7.33 | Med |
| B200 | 192 GB | $4.06 | $3.44–7.45 | High |
| H200 | 141 GB | $3.55 | $1.97–7.89 | High |
| H100 NVL | 94 GB | $2.35 | $1.33–4.67 | Low |
| H100 SXM | 80 GB | $2.00 | $1.47–3.93 | Med |
| RTX PRO 6000 S | 48 GB | $1.39 | $0.67–2.53 | Med |
| RTX PRO 6000 WS | 96 GB | $1.19 | $0.10–2.67 | Med |
| A100 SXM4 | 80 GB | $0.91 | $0.40–2.00 | Med |
| **RTX 5090** | 32 GB | **$0.53** | $0.21–53.33 | **High** |
| L40S | 48 GB | $0.47 | $0.43–1.20 | Low |
| **RTX 4090** | 24 GB | **$0.40** | $0.13–12.00 | **High** |
| RTX A6000 | 48 GB | $0.39 | $0.29–0.67 | Low |
| RTX 3090 | 24 GB | $0.16 | $0.08–1.33 | High |

Pricing tiers on the platform: **on-demand** (headline, guaranteed), **interruptible** (50%+ cheaper, preemptible), **reserved** (1/3/6-month terms, up to 50% off). Host-side: the platform's commission is roughly **25–35% of rental income**, and Vast's own host marketing quotes RTX 5090 host take-home of $0.30–0.60/GPU-hr and a 4×5090 rig at 80% utilization earning **$700–1,400/month**.

## 1.2 The price-decline curve (the single most important underwriting input)

- **RTX 4090** rentals: ~$1.84/hr (mid-2024) → ~$0.97/hr cross-provider median (2026); Vast, the cheapest marketplace, now clears at **$0.40**. The 4090 lost ~30% of its value within 4 months of the 5090 announcement.
- **H100**: ~$8/hr (2023 peak) → ~$2.85–3.50 (late 2025), ~64% off peak; early-2026 average ~$3.11 with spot as low as $1.25. But not monotonic: 1-year contract rates *rose* ~40% (Oct 2025 → Mar 2026, $1.70 → $2.35) on renewed demand.
- Forward expectation: a further **10–20% step-down** as B200-class supply broadens.

**Underwriting assumption used throughout this doc: consumer-card rental prices decline 20%/yr (base) / 30%/yr (stress), with a 5–6-year revenue life per GPU generation.** The deck's flat $0.49/hr over 10 years is not financeable; this curve is.

---

# Part II — Recalculated unit economics (full unit cost, GPUs included)

## 2.1 What the deck assumed (v3.5) vs. what we now use

The deck's per-unit "$34K/yr" implies exactly **8× RTX 4090 at 100% utilization, $0.49/GPU-hr, 8,760 hr/yr** (8 × 8,760 × $0.49 = $34.3K), with electricity ($4.9K/yr at 3.3 kW) treated as the customer's pre-existing heating bill. Its fleet table priced 4090s at **$3,500 each — a 2023-era price for a card that now sells refurbished for ~$1,500–2,000**, and it counted **GPU CAPEX only**, omitting the heater itself and installation.

Three corrections applied below:

1. **Full unit CAPEX** = GPUs + thermal balance-of-system + install (what the user asked for).
2. **Live market price** ($0.40 4090 / $0.53 5090 medians), a **25% platform fee**, and **realistic rental utilization** (60–80% commercial), not 100%.
3. **The thermal-demand constraint made explicit**: electricity is only "free" during hours the host actually needs heat. A single-family home's hot-water load (~4,500 kWh/yr) absorbs only ~16% of the unit's 28,900 kWh/yr maximum thermal output. Compute *can* run outside heat-demand hours, but then it pays retail electricity (~$0.07/GPU-hr for a 4090 at $0.17/kWh) — still margin-positive when rented, but no longer free. **This is why commercial (hotels, apartments, campuses — near-continuous draw) is the financeable asset class, exactly as the deck's "near-term growth focus" slide says.**

## 2.2 Full unit CAPEX build-up

Two equivalent 3.3 kW builds (same thermal spec as H1):

| Component | H1-4090 build (8× RTX 4090 @ ~412 W) | H1-5090 build (6× RTX 5090 @ ~550 W) |
|---|---:|---:|
| GPUs (2026 street/refurb: 4090 ≈ $1,800; 5090 ≈ $2,400) | $14,400 | $14,400 |
| Thermal BOM — tank, heat exchanger, immersion/coldplate loop, pump, power electronics, networking, enclosure, assembly *(assumption — replace with actual BOM)* | $2,500 | $2,500 |
| Installation (plumber + electrician) | $1,100 | $1,100 |
| **Total installed CAPEX** | **$18,000** | **$18,000** |

*(At the deck's stale $3,500/GPU the 4090 build would be $31.6K installed — the GPU price collapse cut unit CAPEX ~43% while cutting revenue only ~18%. Falling GPU prices are a tailwind for deployment economics even as they pressure rental rates.)*

Annual GPU-hours: 4090 build **70,080**; 5090 build **52,560**. Gross revenue at 100% utilization, median on-demand, no fees: **$28.0K** vs **$27.9K** — equivalent. The 5090 build is preferred going forward (newer silicon → slower price decay, 2 fewer cards to service); numbers below use the 4090 build for continuity with the deck.

## 2.3 Recalculated per-unit annual compute economics

Net margin = GPU-hr × utilization × $/hr × (1 − 25% platform fee) − electricity paid for rented hours falling outside heat demand.

| Scenario | Assumptions | Gross billings | Net to owner | vs deck's $34.3K |
|---|---|---:|---:|---:|
| **Deck (v3.5)** | $0.49/hr, 100% util, no fee, no elec | $34.3K | $34.3K | — |
| **Spot-median ceiling** | $0.40/hr, 100% util, no fee | $28.0K | $28.0K | −18% |
| **Commercial high** | 80% util, $0.40, 25% fee, 70% heat-overlap | $22.4K | **$15.6K** | −54% |
| **Commercial base** ★ | 60% util, $0.40, 25% fee, 70% heat-overlap | $16.8K | **$11.7K** | −66% |
| **Residential + space heating** | 12,000 kWh/yr thermal load monetized at 70% + 25% paid-electricity hours | $11.3K | **$8.5K** | −75% |
| **Residential DHW-only** | 4,500 kWh/yr hot-water load only, 70% of those GPU-hrs rented | $3.1K | **$2.3K** | −93% |
| **Stress (for debt sizing)** | 50% util, price −30% ($0.28), 25% fee | $9.8K | **$6.8K** | −80% |

★ = base case for all financing math.

**Honest cross-check against Superheat's own proof point:** the deck reports **$550K+ cumulative passive income across 1,000+ deployed units** — i.e., roughly $300–600/unit/yr realized to date (largely ASIC/bitcoin-era units in residential DHW settings). That is consistent with the *Residential DHW-only* row, not the $34K headline. The financing program below is therefore built on **metered commercial AI-inference cash flows**, and Phase 1 exists precisely to build that loan tape.

## 2.4 Per-unit returns (commercial base case, 20%/yr price decline)

- Year-by-year net cash: $12.0K → $9.6K → $7.7K → $6.1K → $4.9K → $3.9K (6-yr GPU revenue life) = **$44.3K cumulative** on $18K CAPEX.
- **NPV @ 12% discount: +$14.5K per unit. Simple payback: ~18 months. Unlevered IRR: ~45%.**
- The heater shell outlives the GPUs (10-yr life): a **year-5–6 GPU module swap** (modular by design) re-ups the revenue curve for a second cycle at then-current card prices — the structured-finance equivalent of *re-leasing* a data center, and a key collateral covenant in §3.5.

## 2.5 Fleet restatement of the deck's 1,000-GPU table

| | Deck (v3.5) | **Recalculated (2026 live market)** |
|---|---:|---:|
| Units / GPUs | — / 1,000 | 125 units / 1,000 GPUs |
| CAPEX | $3.50M (GPUs only, $3,500/card) | **$2.25M (fully installed, incl. heaters + labor)** |
| Annual compute revenue | $4.29M | $2.10M gross billings (60% util, $0.40) |
| Platform fee (25%) | not modeled | −$0.53M |
| Electricity (non-heat-overlap rented hrs) | $0 | −$0.11M |
| **Net annual cash** | $4.29M | **$1.47M** |
| **Break-even** | **9.8 months** | **~18.4 months** |
| Traditional-cloud comparison (deck: 23.1 mo) | 2.4× faster | still ~1.5–2× faster — the relative advantage survives because a traditional operator faces the same rate collapse *plus* cooling/facility overhead Superheat doesn't pay |

**Bottom line of the recalc:** the structural thesis holds (zero-cooling, dual-use energy beats any centralized operator on cost), but at live prices the *absolute* numbers are roughly **⅓ of deck headline** at realistic utilization — ~$12K/unit-yr commercial, ~18-month payback, ~45% unlevered IRR. Those are still excellent project-finance numbers — comfortably strong enough to lever. That is what Part III does.

---

# Part III — The financing business plan: originate, package, sell debt to the market

## 3.0 Design principle

Split the business along the same line the consumer-finance market already splits:

- **Residential** → the *customer* owns the unit; Superheat (with a lending partner) **originates a consumer loan** at point of sale → sells/securitizes the loans. *(Analog: GoodLeap home-improvement ABS — water heaters and HVAC are literally an existing eligible asset class.)*
- **Commercial** → **Superheat owns the unit** (third-party ownership, TPO); the host signs a long-term heat-services agreement; compute revenue + heat fees fund an SPV that issues asset-backed debt. *(Analogs: Sunrun/Sunnova solar leases; CoreWeave's GPU-backed SPV facilities; data-center ABS.)*

Two asset classes, two debt machines, one servicing/orchestration layer that Superheat never sells.

## 3.1 The funding ladder (sequenced, per a16z's canonical fintech debt progression)

| Rung | Instrument | Size | Cost (indicative) | When |
|---|---|---:|---|---|
| 0 | Equity-funded pilot fleet | $2–5M | equity | Now — build metered loan tape |
| 1 | Venture debt / equipment lines | $5–15M | 10–14% | 6–12 mo |
| 2 | **Warehouse SPV facility** (revolving, asset-backed) | $15–50M | SOFR + 250–450 | 12–24 mo |
| 3 | **Forward flow** (residential loans sold at par as originated) | uncapped (run-rate) | buyer takes asset yield; Superheat keeps fees | 18–30 mo |
| 4 | **Term ABS takeout** (144A) | $150–400M per deal | solar/DC-ABS spreads (Sunrun 2026 priced at ~220 bps) | 24–36 mo |
| 5 | Programmatic issuance + investment-grade corporate path | $0.5–1B+/yr | IG possible — CoreWeave's 2026 $8.5B DDTL rated A3/A(low) | 36 mo+ |

Key mechanics: a **warehouse** is a revolving loan to a bankruptcy-remote SPV that *holds* assets until enough accumulate for a term takeout (Superheat keeps upside, bears performance risk); a **forward flow** is a committed *sale* of assets at par as they're originated (buyer's balance sheet funds growth, buyer takes credit risk, Superheat keeps origination + servicing fees). Run both: forward flow for residential volume, warehouse→ABS for the commercial fleet.

## 3.2 Product A — Residential: the consumer-loan origination machine

**The offer:** "Replace your dying water heater with an H1 at **$6,000–9,000 installed**, financed at ~12.5% APR over 10 years — **$88–132/month** — and your share of the unit's compute income (~50% of net) offsets some-to-all of the payment." DHW-only homes generate ~$2.3K/yr net (≈$96/mo customer share ~ payment-neutral); cold-climate homes with hydronic space heating ~$8.5K/yr (≈$350/mo customer share — *the heater pays the mortgage on itself*). Superheat books **hardware margin ($2–4K/unit) + origination economics** up front.

**Why this is immediately financeable — the GoodLeap template:** GoodLeap Home Improvement Solutions Trust 2026-1 securitized **37,987 loans, $11,960 average balance, 12.56% WAC, 757 weighted-avg FICO** — raising **$408.9M against a $454.3M pool**, Class A credit enhancement 21.89%. An H1 loan is the *same paper* (home-improvement loan, prime homeowner, similar balance, lien/UCC-1 on fixture) **plus an income-producing asset behind it** — a strictly better credit story than a window or HVAC loan, because the collateral generates cash that supports the borrower's payment.

**Execution sequence:**
1. **Partner-originate first** (GoodLeap/Mosaic-style platform or a regional bank): zero balance-sheet risk; Superheat is "manufacturer + contractor," paying a standard dealer fee. Validates take rates and default behavior.
2. **Captive origination** once volume justifies it (state lending licenses or bank-partnership/rent-a-charter): capture the **origination fee (2–5%)**, the **dealer-fee spread**, and the **servicing strip (0.75–1.25%/yr)**.
3. **Forward flow** the whole loans at/above par to 1–2 committed buyers (credit funds, insurers); retain servicing.
4. **Term ABS** under Superheat's own shelf once originations exceed ~$200M/yr; gain-on-sale recognized conservatively (§3.8).

**Per-loan P&L at captive scale** (on a $7,500 loan): origination fee ~$225–375 + dealer-spread/gain-on-sale ~$150–300 + servicing NPV ~$400–600 + hardware margin $2,000–4,000 → **$2.8–5.3K of economics per residential placement**, most of it cash up front, while the loan itself lives on someone else's balance sheet.

## 3.3 Product B — Commercial TPO fleet: the securitizable core

**Structure (the solar-lease playbook with a CoreWeave collateral package):**

```
Hotel / apartment / campus host
        │  10-yr Heat Services Agreement (HSA):
        │  host buys hot water at 10–20% below grid-equivalent cost,
        │  guarantees minimum thermal draw, provides space + meter
        ▼
┌─────────────────────────────────────────────┐
│  SUPERHEAT ASSETCO SPV (bankruptcy-remote)  │
│  owns units; pledges: equipment (UCC-1),    │
│  HSA contracts, compute receivables,        │
│  platform accounts, SPV equity              │
└──────┬──────────────────────────┬───────────┘
       │ compute revenue          │ heat-service fees + DR/VPP income
       ▼                          ▼
   Collection account → waterfall (§3.5) → noteholders → residual to Superheat
```

- **Two cash-flow legs per unit:** (1) compute rental income (variable, ~$12K/yr base) routed through marketplace + direct enterprise inference contracts; (2) **contracted heat-service fees** (small but *fixed* — say $40–80/mo per unit, the host's discount-but-committed heating spend) + demand-response/VPP income (~$20–100/unit-yr; PGE-style programs pay per enrolled water heater, and DOE pegs VPP capacity at ~half the cost of conventional peakers). The fixed legs anchor the rating; the compute leg provides the excess spread.
- **Debt sizing at the unit level:** 75% advance on $18K = $13.5K; 5-yr amortizing at ~9% = $3.47K/yr debt service → **DSCR 3.5× base / 2.0× stressed** (50% util, −30% price). These coverage levels at a 75% advance rate are conservative against solar precedent (Sunrun's 2026 deal: **79.3% advance**) — appropriate for a first-of-kind asset.
- **Precedent that GPUs-as-collateral clears institutional credit committees:** CoreWeave's progression — **$2.3B (2023) → $7.5B (2024, Blackstone/Magnetar-led) → $8.5B DDTL 4.0 (2026) rated A3 / A(low), priced at SOFR+225 floating / ~5.9% fixed**, secured by GPU servers + customer contracts in an SPV — the first investment-grade GPU-backed financing. Superheat's pool argument is *stronger on diversification* (thousands of small obligors vs. one hyperscaler counterparty) and *weaker on contract tenor* (marketplace rates vs. take-or-pay) — which is exactly what the HSA fixed leg and the enhancement stack below compensate for.

## 3.4 Product C — Host equipment financing (optional flywheel accelerant)

Vast.ai already operates **"Vast Finance"** — matching hosts with equipment-as-collateral leases, revenue-based loans, and sale-leasebacks, underwritten on verified platform earnings. Superheat can do the same for its commercial channel partners (franchised installers, building owners who want to own units themselves): Superheat or a captive finance partner lends against the unit + assigned compute earnings, **originating yet another securitizable receivable** while shifting CAPEX off Superheat's balance sheet. Same machine, third asset feed.

## 3.5 The securitization itself — waterfall design (term sheet skeleton)

**Superheat Compute & Thermal Trust 2027-1** (illustrative inaugural deal):

- **Pool:** 10,000 seasoned commercial units (≥12 months metered history), $180M original cost, ~$120M/yr year-1 net collections (base case).
- **Issuance: $150M notes against the pool** (~75–80% effective advance, mirroring Sunrun's 79.3%):

| Class | Size | Rating target | Enhancement | Indicative coupon |
|---|---:|---|---:|---:|
| A (senior) | $105M | A / A− | ~30% subordination + OC | SOFR + 200–250 |
| B (mezz) | $30M | BBB− | ~12% | SOFR + 350–450 |
| C (junior) | $15M | BB | ~4% | SOFR + 600–700 |
| Residual / equity | retained | unrated | — | excess spread to Superheat |

- **Credit enhancement stack** (benchmarked to Sunrun Pangea/Bacchus and Sunnova Sol VI, whose A/B/C enhancement ran 39.1%/26.3%/16.2%): **overcollateralization ~20%** (Pangea 2025-2 used 19.5%), subordination as above, **liquidity reserve** (6 months senior interest), **supplemental GPU-refresh reserve** (funds the year-5 module swap — the novel, Superheat-specific feature; the analog of solar's inverter-replacement reserve), and excess spread trapped on trigger breach.
- **Amortization:** 5–6-year expected note life (matched to GPU revenue life), *not* solar's 20–25 years. Faster amortization is the honest answer to compute-price-decline risk: the pool de-levers ahead of the depreciation curve (20%/yr base decline is built into sizing; DSCR triggers assume it).
- **Performance triggers:** 3-month-average DSCR < 1.5× → turbo amortization (all excess spread pays down Class A); fleet rental utilization < 40% or platform concentration > 60% on any single marketplace → cash trap; servicer-replacement triggers with a named backup servicer (the standard cure for "only Superheat can run this" concentration risk).
- **Collateral covenants unique to this asset:** minimum metered-uptime SLA per unit; GPU-refresh obligation funded from the reserve; geographic and obligor concentration limits; minimum % of revenue from contracted (HSA + reserved-instance) sources rising over the deal's life.

**Who buys what** (mapping to the buyer universe in the original framing): insurance companies and pensions take Class A (yield over treasuries with structural protection); asset managers and structured-credit funds take B; hedge funds/specialty credit take C; **Superheat keeps the residual + the 0.5–2%/yr servicing strip — the permanent, sticky annuity.**

## 3.6 Rating-agency path

The methodology infrastructure now exists — this was *not* true two years ago:

- **KBRA — Data Center ABS Global Rating Methodology (Jan 9, 2026)**; KBRA also rates the GoodLeap home-improvement shelf (residential analog).
- **Moody's — data-center securitization methodology (Feb 2025)**: favors newer assets, strong OC, first-lien trust structures.
- **S&P — first global data-center criteria (June 2024, revised Aug 2025)**; published "ABS Frontiers" on equipping data centers through securitization.
- **Fitch — exposure drafts (June/July 2025)** for single-borrower data-center deals.
- Market depth: **~$57B of US data-center securitization since 2021; ~$26B in 2025 alone ($15B ABS + $11B CMBS); sell-side projects $30–40B/yr through 2027** — an investor base actively hunting for exactly this exposure, currently starved of *diversified* (non-single-campus) deals. A 10,000-site distributed pool is a portfolio-construction gift to that base.

Engagement plan: pre-engage KBRA + one of Moody's/S&P at warehouse stage (Phase 2) with the metered tape; target inaugural rated deal within 12 months of warehouse close.

## 3.7 Stackable contracted revenue (rating fuel)

Each adds a *fixed* leg that improves the contracted-revenue ratio rating agencies reward:

1. **Demand response / VPP enrollment** — $20–100/unit-yr (PGE pays $20/device/yr; co-ops pay $100 sign-on credits; DOE: VPPs deliver peak capacity at ~½ the cost of a gas peaker). At 100K units this is $2–10M/yr of utility-grade contracted income.
2. **Reserved-instance compute** — Vast's reserved tier (1/3/6-month commitments) converts spot exposure into contracted exposure at a known discount; target ≥30% of fleet GPU-hours reserved by deal close.
3. **Heat-service agreements** — the HSA minimum-draw clause is itself a contracted commodity sale (heat-as-a-service), the direct analog of a solar PPA.
4. **Efficiency/electrification incentives** — utility rebates for heat-pump-class water heaters, state electrification programs, and (where applicable) emissions-displacement credits monetized at origination, GoodLeap-style, to compress the net CAPEX basis.

## 3.8 Risk register (what kills deals — name it first)

| Risk | Reality check | Mitigant in structure |
|---|---|---|
| **Compute price decline** | 4090 rentals −~50%/yr on Vast since 2024; H100 −64% from peak | 20–30%/yr decline *in sizing*; 5–6-yr amortization; turbo triggers; GPU-refresh reserve; falling card prices also cut replacement CAPEX 40%+ |
| **Utilization shortfall** | Realized fleet income to date (~$550/unit-yr deck proof point) ≪ modeled commercial base | Underwrite only metered, seasoned units into pools; commercial-only collateral; reserved-instance floor |
| **Platform dependence** | One marketplace's fee/policy change hits the whole pool | Multi-platform routing requirement + direct enterprise contracts; concentration trigger at 60% |
| **Issuance-market freeze** (the 2008/Carvana failure mode) | The flywheel assumes CBS keeps selling | Committed multi-year warehouse + forward-flow capacity ≥ 12 months of origination; growth governed by *committed* capital, never by assumed take-out |
| **Gain-on-sale optics** | Carvana's credibility trap | Recognize conservatively; publish static-pool curves quarterly; retain risk via residual (skin in the game, Reg-RR compliant) |
| **Thermal-demand shortfall** | Free electricity exists only when heat is needed | HSA minimum-draw covenants; commercial-first; tank-as-thermal-battery scheduling to shift heating into rented hours |
| **Servicer concentration** | "Only Superheat can run the fleet" | Backup-servicer agreement + escrowed orchestration runbooks/code; standard esoteric-ABS cure |
| **Obsolescence/refresh execution** | GPU module swap at yr 5 must actually happen | Funded refresh reserve inside the trust; modular hardware design requirement |

## 3.9 Phased roadmap with capital targets

**Phase 1 — Tape building (now → 12 mo).** Deploy 1,000–2,000 *commercial* units ($18–36M, equity + venture debt). Instrument everything: per-unit metered GPU-hours, realized $/hr, utilization, uptime, thermal draw, host payment behavior. Launch residential via a **partner originator** (GoodLeap-style) at zero balance-sheet cost. *Exit criteria: 12 months of clean monthly tape on ≥1,000 units; ≥30% reserved/contracted revenue share.*

**Phase 2 — Warehouse + forward flow (12–24 mo).** Close a **$25–50M revolving warehouse** (bank or private-credit lender, SOFR+250–450, ~70–75% advance) against the seasoned fleet; sign **forward-flow commitments** for residential loan production; pre-engage KBRA/Moody's. Begin demand-response enrollment fleet-wide. *Exit criteria: warehouse ≥70% utilized, DSCR ≥2.5× trailing-6-month, rating-agency pre-read complete.*

**Phase 3 — Inaugural term ABS (24–36 mo).** **$150M Superheat Compute & Thermal Trust 2027-1** as specified in §3.5; recycle proceeds into the next 8,000–10,000 units; servicing strip + residual stay home. *Exit criteria: deal prices inside SOFR+300 blended; advance ≥75%.*

**Phase 4 — Programmatic (36 mo+).** 2–4 deals/yr; residential shelf separate from commercial shelf; pursue the CoreWeave precedent toward **investment-grade** senior classes; international thermal-utility partnerships. At 100K commercial units, the machine self-funds: **~$1.2B/yr net collections supporting ~$1.4B of standing ABS, with Superheat's retained economics (servicing + residual + origination + hardware margin) compounding off other people's balance sheets** — which is the whole point.

---

# One-sentence summary

At live June-2026 GPU prices and honest utilization, a fully-loaded $18K Superheat commercial unit nets ~$12K/yr (≈18-month payback, ~45% unlevered IRR — about a third of the deck's headline but robust and now *underwritable*), and the company's real leverage is to stop selling boxes and start **manufacturing two financial assets** — prime home-improvement loans on the residential side and contracted thermal-compute cash flows on the commercial side — **warehoused, forward-flowed, and securitized through the now-proven GPU-collateral (CoreWeave), solar-lease (Sunrun), and home-improvement (GoodLeap) channels**, while Superheat permanently retains origination fees, the servicing strip, and the residual.

---

# Sources

**GPU market pricing & decline**
- Vast.ai live pricing snapshot: local save of [vast.ai/pricing](https://vast.ai/pricing) (2026-06-11) — table in §1.1
- [Vast.ai — How much can you earn renting your GPU](https://vast.ai/article/how-much-money-can-you-earn-renting-out-your-gpu-on-vast-ai) · [Vast.ai hosting calculator](https://vast.ai/hosting/calculator) · [Vast.ai docs — pricing/fees](https://docs.vast.ai/documentation/instances/pricing)
- [Vast Finance — GPU financing for hosts](https://vast.ai/hosting/finance)
- [Introl — GPU cloud price collapse: H100 −64%](https://introl.com/blog/gpu-cloud-price-collapse-h100-market-december-2025) · [Silicon Data — H100 rental price 2023–2025](https://www.silicondata.com/blog/h100-rental-price-over-time-2023-to-2025-a-complete-market-analysis) · [Thunder Compute — AI GPU rental trends, June 2026](https://www.thundercompute.com/blog/ai-gpu-rental-market-trends) · [Hashrate Index — used A100/H100 market](https://hashrateindex.com/blog/used-gpu-market-pricing-deprecation-secondary-ai/) · [SemiAnalysis — H100 rental index](https://newsletter.semianalysis.com/p/the-great-gpu-shortage-rental-capacity)

**GPU-backed debt precedent**
- [CoreWeave — $8.5B DDTL 4.0, first investment-grade GPU-backed financing (A3/A-low, SOFR+225/5.9%)](https://investors.coreweave.com/news/news-details/2026/CoreWeave-Closes-Landmark-8-5-Billion-Financing-Facility-Achieving-First-Investment-Grade-Rated-GPU-backed-Financing/default.aspx)
- [Blackstone — CoreWeave $7.5B facility](https://www.blackstone.com/news/press/coreweave-secures-7-5-billion-debt-financing-facility-led-by-blackstone-and-magnetar/) · [CoreWeave — $2.3B facility (2023)](https://www.coreweave.com/blog/coreweave-secures-2-3-billion-debt-financing-magnetar-capital-blackstone) · [CoreWeave credit agreement (SPV collateral)](https://contracts.justia.com/companies/coreweave-inc-103508/contract/1316739/)

**Solar & home-improvement ABS templates**
- [Sunrun $584M securitization — 79.3% advance, 220bps](https://investors.sunrun.com/news-events/press-releases/detail/367/sunrun-prices-584-million-securitization-of-residential) · [Sunrun Pangea 2025-2 — 19.5% OC](https://asreport.americanbanker.com/news/sunrun-pangea-seeks-to-raise-481-million-in-residential-solar-abs) · [Sunrun Bacchus 2025-1 — 4 tranches, reserves](https://asreport.americanbanker.com/news/sunrun-bacchus-returns-to-raise-695-million-on-solar-revenue) · [Sunnova Sol VI — A/B/C enhancement 39.1/26.3/16.2%](https://asreport.americanbanker.com/news/sunnova-launches-435-million-in-solar-panel-finance-abs)
- [KBRA — GoodLeap Home Improvement Solutions Trust 2026-1 prelim ratings](https://www.kbra.com/publications/dmtSycxM/kbra-assigns-preliminary-ratings-to-goodleap-home-improvement-solutions-trust-2026-1) · [ASR — GoodLeap $408.9M home-improvement ABS](https://asreport.americanbanker.com/news/goodleap-raises-408-9-million-in-abs-from-home-improvement-portfolio) · [GoodLeap $523M securitization](https://pv-magazine-usa.com/2025/12/15/sustainable-home-upgrade-finance-platform-goodleap-announces-523-million-securitization/)

**Data-center ABS market & rating criteria**
- [GlobalCapital — US data center ABS in 2026 and beyond](https://www.globalcapital.com/article/2fph3tnpn4zkkckgulyio/securitization/abs-us/too-big-to-ignore-us-data-center-abs-in-2026-and-beyond) · [CRA Insights — Data center ABS risks/yields/ratings (Dec 2025)](https://media.crai.com/wp-content/uploads/2025/12/03163400/Insights-Data-center-ABS-%E2%80%93-Risks-yields-and-ratings-December2025.pdf) · [S&P — ABS Frontiers: equipping data centers](https://www.spglobal.com/ratings/en/regulatory/article/abs-frontiers-equipping-data-centers-through-securitization-s101645975) · [Fitch — proposed data-center criteria](https://asreport.americanbanker.com/news/fitch-ratings-proposes-new-criteria-to-rate-data-center-securitizations) · [RBC — understanding data center securitization](https://www.rbccm.com/en/insights/2025/12/the-infrastructure-revolution-understanding-data-center-securitisation) · [CREFC — Data Center e-Primer (Feb 2026, incl. KBRA methodology date)](https://www.crefc.org/common/Uploaded%20files/Learn/DataCenters-eprimer-2026.02.25.pdf)

**Funding-structure mechanics**
- [a16z — Launching a financial product: choosing a funding structure](https://a16z.com/launching-a-financial-product-how-to-choose-the-right-funding-structure/) · [a16z — Negotiating your first warehouse facility](https://a16z.com/negotiating-your-first-warehouse-facility-trade-offs-across-terms/) · [a16z — Fintech guide to raising debt](https://d1lamhf6l6yk6d.cloudfront.net/uploads/2023/10/a16z-Fintech-Debt-Raise-Guide-Updated.pdf)
- [Cascade — Forward flow vs warehouse](https://www.cascadedebt.com/insights/forward-flow-vs-warehouse-facilities-understanding-your-options) · [Finley — What is forward flow](https://www.finleycms.com/blog/what-is-forward-flow) · [Weil — Forward flow transactions (structured finance talking points)](https://www.weil.com/-/media/files/pdfs/2024/may/forward-flow-transactions.pdf) · [Hogan Lovells — the appeal of forward flow](https://www.hoganlovells.com/en/publications/going-with-the-flow-the-appeal-of-the-forward-flow-transaction-in-structured-finance) · [DLA Piper — forward flow arrangements](https://www.dlapiper.com/en/insights/publications/2018/10/finance-and-markets-global-insight-issue-15/forward-flow-arrangements)

**Grid-services / VPP revenue**
- [Portland General Electric — connected water heaters ($20/device-yr)](https://portlandgeneral.com/property-managers/connected-water-heaters) · [DOE — Virtual power plant projects (VPP ≈ ½ cost of peaker)](https://www.energy.gov/edf/virtual-power-plants-projects) · [Virtual Peaker — water heaters in demand response](https://virtual-peaker.com/blog/how-are-water-heaters-used-for-demand-response/) · [Rheem — VPPs and water heating](https://www.rheem.com/water-heating/articles/virtual-power-plants-the-newest-way-to-save-energy-and-money/)

*Assumptions flagged inline (thermal BOM $2,500, install $1,100, platform fee 25%, heat-overlap 70%, decline 20%/yr base) should be replaced with actuals as metered data accumulates. Securitization structures herein are proposed, not executed; validate with structuring counsel and an ABS bookrunner.*
