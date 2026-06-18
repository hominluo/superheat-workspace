# Superheat — Financial Model, Build Roadmap & Risk Register

**Status:** Decision-input · **Date:** June 2026 · **Audience:** Founders + underwriting + structuring counsel + engineering leads + program management + rating-agency pre-read.

**Purpose.** This document is the single financial-and-execution artifact for the commercial Superheat unit. It consolidates three previously separate memos into one coherent narrative:

- **Part A — CAPEX reconciliation.** Turns the two coupled open questions — *what is the true commercial CAPEX?* and *8 vs 12 GPUs per commercial unit?* — into a single decision, restating the market memo's $18K to **~$20–22K** and re-running payback/IRR/DSCR on the honest number. This is the number the securitization advance rate is struck against, so it must be firm before any financing.
- **Part B — OPEX model & financial waterfall.** Supplies the **missing third leg** of the unit model: a per-unit/fleet OPEX line and a waterfall that turns the metered tape into a defensible NET margin, then corrects the IRR/DSCR/payback figures (declining-stream method, financing assumptions, utilization sensitivity, tape linkage).
- **Part C — Build roadmap.** Expands the build-phasing and open-questions sections of `system-design-overview.md` into an executable engineering roadmap whose technical exit criteria are exactly the financial entry criteria for each rung of the funding ladder.
- **Part D — Consolidated risk register (R1–R15).** Pulls every open question and risk distributed across the design docs and market-research risk sections into one engineering-owned register.

**The single consistent financial picture carried throughout this doc:** installed CAPEX **~$20–22K** (8× RTX 4090); canonical declining net stream **$12.0 → $9.6 → $7.7 → $6.1 → $4.9 → $3.9K** (Σ $44.3K); payback **~22/25 mo pre-OPEX**, **~29/33 mo post-OPEX-and-reserve**; post-OPEX base DSCR **~2.3–2.6×**, stress **~1.1–1.3×**; financing **75% advance, ~9% all-in, 5-yr amortization**. These numbers are stated once, consistently, and the CAPEX restatement (Part A) feeds directly into the OPEX waterfall (Part B).

> **TL;DR (whole doc).** The $18K market-memo CAPEX omits a real 8-GPU compute host (~$3.5–4.5K); true installed CAPEX is **~$20–22K**. Layering on a per-unit OPEX of **~$1,130/unit-yr additive cash (~$0.027/GPU-hr) + ~$1,000/unit-yr refresh reserve**, the underwriting cash flow falls to **~$9,870 in year 1 (~18% below the memo's net)**. Run honestly against the **declining** stream, payback is **~22–25 mo pre-OPEX / ~29–33 mo post-OPEX-and-reserve**, post-OPEX base DSCR **~2.3–2.6×**, post-OPEX stress DSCR **~1.1–1.3×**. The asset is still strongly financeable, but the dominant risk to net margin is **utilization × overlap (the 60%×70% assumption)**, not OPEX and not the $4K host. **Recommendation: freeze the SKU at 8× RTX 4090, restate CAPEX to ~$21K, re-run the entire waterfall on net-of-OPEX, and carry the post-OPEX figures into investor and rating-agency materials.**

---

# Part A — Commercial Unit CAPEX Reconciliation & Decision Memo

## A1. The two questions are coupled

You cannot answer "what does a commercial unit cost" without first fixing "how many GPUs is a commercial unit." The pitch says **12 GPUs / 33 kW**; the economics memo underwrites **8 GPUs / 3.3 kW** and *every* financing number (CAPEX, payback, IRR, DSCR, ABS pool sizing) rests on the 8-GPU figure. `node.md` recommends freezing at **8×** for four reasons (matches the underwritten tape, 3.3 kW is a real single-tank thermal envelope, 8 maps to commodity host platforms, and thermal-overlap risk scales with GPU power). This part adopts that and prices the 8-GPU unit honestly. The SKU freeze is also start-blocking risk **R3** (Part D).

## A2. The reconciliation — what the $18K leaves out

The memo's build-up is `8 × $1,800 GPU ($14,400) + $2,500 "thermal BOM" + $1,100 install = $18,000`. The gap: an 8-GPU always-on multi-tenant **compute host** — CPU(s), server board, 256 GB ECC RAM, an 8–16 TB NVMe array, redundant PSUs, NIC + cellular failover, BMC — is **not** covered by a $2,500 line that is described as tank/heat-exchanger/pump/electronics/enclosure. At fleet scale that host is realistically **$3,500–4,500**.

### A2.1 CAPEX bridge table

| Line item | Memo (as written) | Reconciled (honest 8-GPU build) |
|---|---:|---:|
| 8× RTX 4090 (refurb @ $1,800) | $14,400 | $14,400 |
| **Compute host** (CPU, board, 256 GB ECC, 8–16 TB NVMe, 2× redundant PSU, NIC, **cellular failover**, BMC) | *(not itemized)* | **$4,000** |
| Thermal balance-of-system (tank, heat exchanger, coldplate/immersion loop, pump, power electronics, enclosure) | $2,500 *(appears to absorb everything non-GPU)* | $2,500 |
| Installation (plumber + electrician) | $1,100 | $1,100 |
| **Total installed CAPEX** | **$18,000** | **~$22,000** (range **$20–22K**) |

The fix is bookkeeping, not a redesign: **split the single $2,500 line into "compute host (~$4,000)" + "thermal balance-of-system (~$2,500)"** and restate the total. The $18K was never a 12-GPU number nor a fully-loaded 8-GPU number — it was an 8-GPU number missing the host.

## A3. Restated unit economics (8× 4090 @ ~$21K)

Holding the memo's revenue assumptions constant (60% utilization × 70% thermal overlap, 25% platform fee → ~$11.7K year-1 net, declining ~20%/yr over a 6-year GPU life: ≈ $12.0 → $9.6 → $7.7 → $6.1 → $4.9 → $3.9K, cumulative ~$44.3K), only the initial outlay changes:

*(The ~$11.7K overview headline and the $12.0K that starts this stream are the **same base case** — the $0.3K difference is rounding, not a different assumption.)*

### A3.1 Restated economics table

| Metric | Memo @ $18K | Reconciled @ $20K | Reconciled @ $22K |
|---|---:|---:|---:|
| Payback (declining stream, pre-OPEX) | ~19 mo | ~22 mo | ~25 mo |
| Unlevered IRR (6-yr, pre-OPEX) | ~45% | ~37% | ~33% |
| Base DSCR (pre-OPEX) | 3.5× | ~3.2× | ~3.0× |
| Stress DSCR (pre-OPEX) | 2.0× | ~1.8× | ~1.7× |
| Cumulative net / CAPEX | 2.46× | 2.22× | 2.01× |

*Method: IRR solves NPV=0 on the declining 6-yr cash-flow stream above. **Payback is computed against that same declining stream via cumulative cash flow** (cum. net ≈ $12.0K after Y1, $21.6K after Y2, $29.3K after Y3): $18K is recovered ~1.6 yr in (~19 mo), $20K ~1.83 yr (~22 mo), $22K ~2.05 yr (~25 mo). The earlier "simple payback = CAPEX / $11.7K flat" approximation overstated recovery speed (it ignored the ~20%/yr decline) and is superseded by the figures above.*

*The DSCR figures in this table are **pre-OPEX and illustrative** — they convey direction and rough magnitude only. DSCR = NOI / debt service, and debt service depends on advance rate, interest spread, and amortization. The real, derived waterfall (with explicit financing assumptions — ~75% advance, ~9% all-in, 5-yr amortization) and the honest **post-OPEX** DSCRs live in **Part B (§B4)**. The Part-A figures are directional, not audited.*

**Read:** the asset is still strongly financeable — ~2× cash-on-cash over the GPU life, sub-2-year (pre-OPEX) payback, IRR in the low-to-mid 30s, DSCR comfortably above 1.0× even stressed. But **pre-OPEX stress DSCR slipping toward ~1.7×** is the line underwriting will care about, and Part B shows the *post-OPEX* stress DSCR is materially thinner still (~1.1–1.3×). It must be modeled honestly, not discovered after a deal prices.

> **Two caveats on what this restatement does *not* fix.**
> - **Revenue is the bigger risk.** Part A only corrects the **cost** side. The **revenue** side (60% utilization × 70% thermal overlap, held constant above) is the more fragile input and the larger DSCR risk — a few points of overlap or utilization shortfall moves the NOI, and therefore DSCR, more than the $18K→$22K CAPEX swing does. See the utilization sensitivity in **§B5** and the fleet-blend reconciliation of the 60%×70% assumption in `platform.md`.
> - **OPEX is not in this part.** Net margin and DSCR also depend on per-unit OPEX — relay/bandwidth, cellular failover, cache, cloud, and payout fees — covered in **Part B**. The DSCR figures above are pre-OPEX and tighten once OPEX is netted (**§B3–§B4**).

## A4. What changes if the business insists on 12 GPUs

Not a BOM tweak — a full re-underwrite. 12× 4090 ≈ 5.4 kW GPU heat is a *different* thermal/electrical/mechanical class (likely 208 V 3-phase, larger/multiple tanks or buffer, exotic PCIe riser/PLX topology, higher committed minimum heat draw to stay thermally overlapped). It raises GPU CAPEX to ~$21.6K *before* a heavier host, chassis, and 3-phase install — plausibly **$28–32K all-in** — and it concentrates the thermal-overlap risk the risk register names (R3, R5). If 12 is chosen for the "fewer sites for 1 GW" narrative, then Slide 8 and the entire waterfall must be restated at 12×, and the 12-GPU "XL" tagged as a **distinct collateral class** in the tape (its own CAPEX basis, DSCR, overlap assumption). Recommendation stands: **launch and finance on 8×; treat 12× as a separately-underwritten future XL SKU.**

## A5. Impact on the securitization tape

The metering ledger is the securitization tape (`platform.md`), and the **advance rate** (how many dollars of notes per dollar of collateral) is struck against the cost basis. Consequences of moving $18K → ~$21K:

- **Cost-basis line per unit** changes from $18K to ~$21K → ~$210M original cost for a 10,000-unit pool (vs $180M in the onepager's illustrative *Trust 2027-1*). Note issuance at a ~75–80% advance scales accordingly.
- **Collateral homogeneity preserved** only if the SKU is frozen at 8× — mixing 8× and 12× units pollutes the pool. This is why the SKU freeze must precede origination at scale.
- **DSCR covenant headroom** is the sensitive output (§A3, §B4). Re-run the stress case at ~$21K before pre-engaging rating agencies (Phase B), not during.

## A6. Part A recommendation & required actions

1. **Freeze the commercial SKU at 8× RTX 4090 (~3.3 kW GPU).** Ratify before hardware freeze. (`node.md`; resolves R3)
2. **Restate commercial CAPEX to ~$21K** (split the BOM line per §A2) and **re-run payback/IRR/DSCR and the waterfall** at $20K and $22K bookends (Part B).
3. **Validate two procurement assumptions that swing the whole number:** a sustainable refurb-4090 channel at ~$1,800 for 10,000+ cards, and the real 8-GPU host cost at volume. (`node.md`; refurb-supply risk in Part D)
4. **Defer 12× to a separately-underwritten XL SKU**; do not let it into a financed pool without its own tape line.
5. **Carry the honest number into investor materials** — Slide 8's $18K/$11.7K/18-mo/45% should be footnoted or restated. The asset survives the restatement; an investor discovering the gap during diligence is the avoidable harm.

---

# Part B — Consolidated OPEX Model & Unit/Fleet Financial Waterfall

This part supplies the per-unit/fleet OPEX line and the waterfall that turns the metered tape into a defensible NET margin, and corrects the IRR/DSCR/payback numbers that are "directional" in Part A. CAPEX is **not** restated here — it carries forward from Part A (~$20–22K). Revenue carries forward from `../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md` (~$11.7K yr-1 net, declining).

> **TL;DR (Part B).** A commercial 8× 4090 unit carries **~$1,130/unit-yr of additive cash OPEX (~$0.027/GPU-hr)** plus a **~$1,000/unit-yr GPU-refresh reserve contribution (~$0.024/GPU-hr)** — together **~$2,130/unit-yr (~$0.051/GPU-hr)**, ~18–19% of yr-1 net revenue. Against the canonical declining stream, the unit recovers **$20K CAPEX at ≈ month 22 (pre-OPEX) / ≈ month 29 (post-OPEX-and-reserve)** and **$22K at ≈ month 25 / ≈ month 33** — NOT the ~20–22 mo a flat-$11.7K assumption gives. DSCR depends entirely on the **financing terms** (75% advance, ~9% all-in, 5-yr amortizing) — it does *not* "scale inversely with CAPEX." The dominant risk to net margin is **utilization × overlap falling** (the 60%×70% assumption), not OPEX and not the $4K host.

## B1. Scope, conventions, and the canonical inputs

**Unit under analysis:** commercial **8× RTX 4090** SKU (`COMM-4090x8-v1`), the financeable collateral class. Frozen per Part A (§A1); 12× is a separately-underwritten XL SKU and out of scope here.

**Canonical inputs held constant from the market memo (every one is an assumption — marked `[A]`):**

| Symbol | Value | Source | Note |
|---|---:|---|---|
| GPUs/unit | 8 | Part A §A1 | frozen SKU |
| GPU TDP | 412 W | `platform.md` (`sku`) | RTX 4090 |
| Installed GPU-hours/yr | **70,080** `[A]` | econ memo §2.2 | 8 × 8,760 |
| Utilization (commercial base) | **60%** `[A]` | econ memo §2.3 ★ | demand-limited; defended by L1 filler (`platform.md`) |
| Thermal overlap (commercial base) | **70%** `[A]` | econ memo §2.3 ★ | free-electricity fraction |
| Rented GPU-hours/yr (yr-1) | **42,048** | = 70,080 × 0.60 | the metered billable base |
| Realized $/GPU-hr (yr-1) | **$0.40** `[A]` | Vast median, econ §1.1 | gross, before platform fee |
| Platform fee | **25%** `[A]` | econ §2.3 | marketplace commission |
| Retail electricity | **$0.17/kWh** `[A]` → **~$0.07/GPU-hr** | `platform.md` | paid only on non-overlap rented hours |
| Price-decline (base / stress) | **20% / 30% per yr** `[A]` | econ §1.2 | the single most important underwriting input |
| GPU revenue life | **6 yr** `[A]` | econ §2.4 | then year-5/6 module swap |
| Installed CAPEX | **$20K (low) / $22K (high)** | Part A §A2–A3 | bookends; replaces stale $18K |

**Canonical 6-year DECLINING net-revenue stream** (econ §2.4, the stream all payback/IRR/DSCR MUST run against):

```
Year     1      2      3      4      5      6     Σ
Net $K  12.0    9.6    7.7    6.1    4.9    3.9   44.3   ← "net" = gross billings − 25% fee − non-overlap electricity, BEFORE the OPEX in §B2
```

> **Method warning carried into every table below.** Payback against a **flat $11.7K/yr** wrongly gives ~$20K ÷ $11.7K ≈ 1.7yr ≈ **20–22 mo on luck**; but the stream *declines*, so the back half of year 2 earns far less than year 1. Computed against the declining stream, $20K recovers **≈ month 22** pre-OPEX / **≈ month 29** post-OPEX-and-reserve, and $22K **≈ month 25** / **≈ month 33** (§B3.1, §B4). Using the flat rate is the error this part exists to stop.

**Two definitions of "net" — do not conflate them:**
- **Net-to-stream (memo's "net"):** gross − 25% platform fee − non-overlap electricity. This is the $12.0K yr-1 figure. It is what the econ memo and the `securitization_tape` view (`platform.md`) call net.
- **Net-of-OPEX (this part's contribution):** net-to-stream − the consolidated OPEX in §B2. This is the number underwriting actually sizes debt against and is **~$1,130–2,130/unit-yr lower** than the memo's net. §B6 wires it into the tape.

## B2. Consolidated per-unit / fleet OPEX model

Every line below is either cross-referenced to an existing design doc that owns the cost driver, or marked `[A]` as a fresh assumption to be replaced with metered actuals. Per-GPU-hr is computed on the **yr-1 rented base of 42,048 GPU-hr/unit-yr** unless noted (idle-driven costs noted as such).

### B2.1 Line-item table ($/unit-yr, $/GPU-hr, fleet-scaling)

| # | OPEX line | $/unit-yr | $/GPU-hr | Owner doc | Scaling with fleet size |
|---|---|---:|---:|---|---|
| 1 | **Relay bandwidth (DERP/TURN)** | $130 | $0.0031 | `connectivity-and-data.md` | **Sub-linear** — fixed POP cost amortizes; per-unit *falls* as cache hit-rate rises. See §B2.2. |
| 2 | **Cellular failover** (commercial LTE/5G backup) | $300 `[A]` | $0.0071 | `connectivity-and-data.md`, Part A host BOM | **Linear** — per-SIM data plan; commercial-only line. Mostly standby; bursts are relay-priced. |
| 3 | **Regional cache / PoP + egress** | $180 | $0.0043 | `connectivity-and-data.md` | **Strongly sub-linear** — content-addressed dedup: one origin pull per region serves thousands of nodes. Per-unit *drops* with density. See §B2.2. |
| 4 | **Control-plane cloud** (compute, queues, Redis, edge aggregators) | $90 | $0.0021 | `platform.md` | **Sub-linear** — shared infra; edge aggregation keeps marginal node cost tiny. |
| 5 | **Time-series DB** (telemetry hot/warm/cold) | $35 | $0.0008 | `platform.md` | **Sub-linear** — per-node frames are small; downsampling + retention tiers bound it. |
| 6 | **Managed Postgres (registry + ledger)** | $20 | $0.0005 | `platform.md` | **Sub-linear** — low write-rate per node; transactional store. |
| 7 | **Payout / settlement rails** (Stripe Connect / ACH) | $60 `[A]` | $0.0014 | `platform.md` | **Linear** — per-payout fee (~$0.25–0.50 ACH or ~0.25%+$0.25 Connect) × ~12 settlements/yr; per-unit roughly flat. |
| 8 | **Monitoring / observability / on-call** (Grafana, alerting, SRE allocation) | $45 `[A]` | $0.0011 | `operations.md` | **Sub-linear** — tooling fixed; on-call scales with incidents, not node count, until very large fleets. |
| 9 | **Field-service / RMA amortization** (truck-roll, swap labor, spares) | $160 `[A]` | $0.0038 | R4 (Part D); `node.md`, `operations.md` | **~Linear** — physical visits scale with units; partly offset by installer-network density and remote-recovery ladder. |
| 10 | **Insurance** (equipment, liability, host-property) | $110 `[A]` | $0.0026 | econ §3 (implied) | **~Linear**, mild volume discount; per-asset premium on ~$21K basis. |
| 11 | **Electricity for non-overlap rented hours** | $883 | $0.0210 | `platform.md` | **Linear** — physical per-unit cost; see §B2.3 (it is NOT free and NOT in the memo's OPEX). |
| | **Subtotal — cash OPEX** | **$2,013** | **$0.0479** | | |
| | *of which non-electricity cash OPEX* | *$1,130* | *$0.0269* | | |
| 12 | **GPU-refresh reserve contribution** (year-5/6 module swap sinking fund) | **$1,000** `[A]` | $0.0238 | econ §2.4, §3.5; R4 (Part D) | **Linear** — per-asset sinking fund; this is the ABS refresh-reserve analog of solar's inverter reserve. Reserve, not a cash cost — see §B2.4. |
| | **TOTAL loaded OPEX (cash + reserve)** | **$3,013** | **$0.0717** | | |

> **Reading the subtotals.** The **non-electricity cash OPEX is only ~$1,130/unit-yr (~$0.027/GPU-hr)** — the platform/infra/service burden is genuinely small, *because the design bought the boring parts and engineered relay/cache cost down* (`connectivity-and-data.md`). The two big lines are **non-overlap electricity ($883)** and the **refresh reserve ($1,000)** — and both are functions of the *revenue/utilization* assumptions, not of infra. That is the headline: **OPEX risk is utilization risk wearing an OPEX coat** (§B5 sensitivity).

### B2.2 Why relay + cache (lines 1, 3) are the cited ~$54K/mo example divided correctly

`connectivity-and-data.md` worked a **$54.4K/mo relay bill for a 5,000-node fleet** — but that is the *pre-mitigation, bulk-on-relay* number. Annualized and per-unit that would be `$54,400 × 12 ÷ 5,000 ≈ $130/unit-yr` **if bulk rode the relay**. The whole point of the cache serving the fast down-link plus shipping checkpoint deltas (not full state) is to **collapse `B` for relayed sessions to control-only traffic (KB/s, not MB/s)**, cutting that by ~an order of magnitude. So:

```
connectivity raw (bulk-on-relay):           ~$130/unit-yr   ← the headline scare number
post-mitigation (bulk-off-relay, line 1):   ~$13/unit-yr    ← residual control/console relay
this doc budgets line 1 = $130/unit-yr CONSERVATIVELY       ← assume mitigation only partially lands early;
                                                              falls toward $13 as cache hit-rate matures
line 3 (cache/PoP/egress) = $180/unit-yr    ← the cost of MOVING bulk off the relay (it lands here instead)
```

`[A]` Lines 1 + 3 together (~$310/unit-yr, ~$0.0074/GPU-hr) are the **R2 opex line** the risk register tracks. They are **sub-linear**: at 50,000 nodes the per-unit figure should be materially lower because POP fixed cost and origin pulls amortize over far more units (origin-warming: one pull per region). **This is the one OPEX cluster that improves with scale** — model it as decaying, not flat.

### B2.3 Why electricity (line 11) is a real OPEX line the memo half-buries

The econ memo treats overlap-hour electricity as the host's pre-existing heating bill (correctly — it's free to Superheat). But the **non-overlap rented hours pay retail** (`platform.md`): `0.60 × 0.30 = 0.18` of clock-hours are rented-but-paid-electricity.

```
non-overlap rented GPU-hr/unit-yr = 70,080 × 0.60 × 0.30 = 12,614 GPU-hr
electricity per GPU-hr at 412 W, $0.17/kWh = 0.412 × 0.17 ≈ $0.070/GPU-hr
non-overlap electricity = 12,614 × $0.070 ≈ $883/unit-yr      [line 11]
```

The memo *does* net this into its $12.0K stream (its "net" subtracts "electricity paid for rented hours outside heat demand"). **So line 11 is already inside the canonical declining stream and must NOT be double-counted in the waterfall (§B3).** It is listed in §B2.1 for completeness and because it is the single most utilization-sensitive line (§B5). **In the §B3 waterfall, electricity sits above the "net" line, not below it.** The cash OPEX that is *additive* to the memo's net is therefore **lines 1–10 = $1,130/unit-yr**, plus the reserve (line 12).

### B2.4 Reserve vs cash (line 12)

The GPU-refresh reserve (line 12) is a **sinking-fund contribution**, not an operating cash outflow — it accrues in the SPV (econ §3.5 "supplemental GPU-refresh reserve") to fund the year-5/6 module swap (R4, Part D). It reduces *distributable* cash but is recovered as a re-upped revenue curve. We carry it at **$1,000/unit-yr `[A]`** (≈ funding a ~$5–6K refurb 8-GPU module swap over 5–6 years; falling card prices, econ §3.8, make this conservative). Underwriting treats it as **structurally senior** (trapped before residual), so the waterfall shows it explicitly (§B3).

### B2.5 Fleet-level OPEX (illustrative, 10,000-unit pool = the Trust 2027-1 pool)

| Line cluster | Per-unit-yr | 10,000-unit fleet/yr | Scaling note |
|---|---:|---:|---|
| Connectivity (relay + cellular + cache/PoP) — lines 1,2,3 | $610 | $6.1M | sub-linear except cellular; model down-trend |
| Cloud + TSDB + Postgres + monitoring — lines 4,5,6,8 | $190 | $1.9M | strongly sub-linear |
| Payout rails — line 7 | $60 | $0.6M | linear |
| Field-service / RMA — line 9 | $160 | $1.6M | ~linear |
| Insurance — line 10 | $110 | $1.1M | ~linear |
| **Cash OPEX additive to memo net (lines 1–10)** | **$1,130** | **$11.3M** | |
| Non-overlap electricity — line 11 *(already in memo net)* | $883 | $8.8M | linear; sits above net |
| **Refresh reserve — line 12** | $1,000 | $10.0M | linear; trapped in SPV |

> **Fleet read:** of ~$11.3M/yr additive cash OPEX on a 10,000-unit pool, **~$6.1M is connectivity** and is the line most likely to *fall* with scale and cache maturation; the rest (~$5.2M) is roughly linear field/insurance/payout/cloud. Against the pool's ~$120M/yr yr-1 net collections (econ §3.5), additive cash OPEX is **~9.4%** of net — small, which is why **the tape's net margin is governed by the revenue assumptions, not the OPEX assumptions** (§B5).

## B3. Corrected unit-economics WATERFALL

One unit, **year 1**, commercial base case (60% × 70%, $0.40, 25% fee). Arithmetic explicit; reconciles to the memo's $12.0K net.

```
GROSS BILLINGS                                                              $/unit-yr
  rented GPU-hr × realized $/GPU-hr = 42,048 × $0.40                         $16,819   [≈ memo §2.3 "$16.8K gross"]
  − Platform fee (25%)              = 16,819 × 0.25                          −$4,205
                                                                            --------
  = Gross after platform fee                                                 $12,614
  − Non-overlap electricity (line 11; 12,614 GPU-hr × $0.070)               −$883
                                                                            --------
  = NET-TO-STREAM  ("net" as the memo & tape define it)                     ≈$12,000   ★ canonical yr-1 ($12.0K)
  ─────────────────────────────────────────────────────────────────────────────────
  − Cash OPEX additive to net (§B2.1 lines 1–10)                            −$1,130
       relay $130 · cellular $300 · cache/PoP $180 · cloud $90 · TSDB $35
       · Postgres $20 · payout $60 · monitoring $45 · field/RMA $160 · insurance $110
                                                                            --------
  = NET-OF-CASH-OPEX (distributable before reserve & debt)                   $10,870
  − GPU-refresh reserve contribution (§B2.1 line 12)                        −$1,000
                                                                            --------
  = NET-OF-OPEX-AND-RESERVE  (the underwriting cash flow)                    $9,870   ◄ what DSCR/owner-split run on
  ─────────────────────────────────────────────────────────────────────────────────
  OWNER SETTLEMENT (commercial TPO: Superheat owns the unit)
     In the TPO/SPV structure (econ §3.3) Superheat IS the owner; the four-party
     "owner settlement" is the host HSA fee, a CONTRACTED COST already inside net
     via the heat-services discount, NOT a residual split. For a RESIDENTIAL unit
     the owner keeps ~50% of net (econ §3.2) — modeled separately, not here.
  ─────────────────────────────────────────────────────────────────────────────────
  SUPERHEAT MARGIN (commercial, pre-debt) = NET-OF-OPEX-AND-RESERVE          $9,870
     This is the SPV's distributable cash that services debt (§B4 DSCR) then
     drops to residual/equity (econ §3.5 waterfall).
```

**Reconciliation note:** the memo's $12.0K "net" is **net-to-stream** (after fee + electricity, before the §B2 cash OPEX and reserve). This part's contribution is the **−$1,130 cash OPEX and −$1,000 reserve** that the memo never modeled, taking the true underwriting cash flow to **~$9,870 in year 1**, i.e. **~18% below the memo's headline net.** Every downstream IRR/DSCR/payback below uses **$9,870** as year-1 and applies the same proportional OPEX/reserve to the declining stream.

### B3.1 Cumulative-cashflow table (post-OPEX, declining stream, $20K & $22K CAPEX)

OPEX + reserve scale **with revenue** (relay/cache/electricity fall as utilization-driven revenue falls; we hold field/insurance/cloud roughly flat). Net-of-OPEX-and-reserve per year, derived as `net-to-stream − cash OPEX − reserve`, with cash OPEX declining ~12%/yr (revenue-linked lines) and fixed lines (~$435: field, insurance, payout, monitoring, Postgres) held flat:

| Year | Net-to-stream (memo) | − Cash OPEX | − Reserve | **= Net-of-OPEX** | Cum. net-of-OPEX | Cum. vs $20K | Cum. vs $22K |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | $12,000 | $1,130 | $1,000 | **$9,870** | $9,870 | −$10,130 | −$12,130 |
| 2 | $9,600 | $1,010 | $1,000 | **$7,590** | $17,460 | −$2,540 | −$4,540 |
| 3 | $7,700 | $910 | $1,000 | **$5,790** | $23,250 | **+$3,250** | +$1,250 |
| 4 | $6,100 | $830 | $1,000 | **$4,270** | $27,520 | +$7,520 | +$5,520 |
| 5 | $4,900 | $760 | $1,000 | **$3,140** | $30,660 | +$10,660 | +$8,660 |
| 6 | $3,900 | $700 | $1,000 | **$2,200** | $32,860 | +$12,860 | +$10,860 |
| | $44,300 | $5,340 | $6,000 | **$32,960 cum.** | | | |

**Payback (post-OPEX, against the DECLINING stream — the corrected numbers):**

```
$20K CAPEX:  recovered between end-yr2 (cum $17,460) and end-yr3 (cum $23,250).
             shortfall at end-yr2 = $2,540; yr3 net-of-OPEX = $5,790 → $5,790/12 = $483/mo
             months into yr3 = $2,540 ÷ $483 ≈ 5.3 → PAYBACK ≈ 24 + 5.3 ≈ month 29  (post-OPEX)
$22K CAPEX:  shortfall at end-yr2 = $4,540 → $4,540 ÷ $483 ≈ 9.4 → PAYBACK ≈ month 33  (post-OPEX)
```

> **Two payback numbers — state both, never just one.**
> - **Pre-OPEX, against the declining stream** (Part A §A3 method, net-to-stream only): $20K ≈ **month 22**, $22K ≈ **month 25**. These are the numbers Part A quotes and they are internally consistent with this part.
> - **Post-OPEX, against the declining stream** (this part, net-of-OPEX-and-reserve): $20K ≈ **month 29**, $22K ≈ **month 33**.
> The flat-$11.7K shortcut (~20–22mo) is wrong against *both* — it ignores the decline. **The honest range to carry into investor materials is ~22–25 mo pre-OPEX / ~29–33 mo post-OPEX-and-reserve.** Underwriting will use the post-OPEX number; marketing the pre-OPEX number without the footnote is the avoidable harm Part A §A6 warns about.

## B4. IRR / DSCR done properly

### B4.1 The financing assumptions DSCR actually depends on (it does NOT "scale inversely with CAPEX")

Part A §A3's footnote treats DSCR as "scaled inversely to CAPEX at a constant advance rate." **That is a modeling shortcut, not how DSCR works.** DSCR is:

```
DSCR (period) = Net cash available for debt service (NCADS)  ÷  Debt service (interest + principal)

where Debt service is set by:  advance rate × CAPEX = note balance (the loan)
                               coupon/spread (SOFR + bps)        = interest
                               amortization schedule (yrs)       = principal
```

CAPEX enters **only** through the loan balance (`advance × CAPEX`). At a **constant advance rate**, a higher CAPEX raises *both* the asset basis *and* the loan proportionally — so debt service rises ~proportionally and DSCR moves *less* than "inversely." DSCR's real sensitivities, in order: **(1) NCADS = net-of-OPEX cash flow (§B3), (2) advance rate, (3) coupon, (4) amortization tenor.** Stating it as "inverse to CAPEX" hides the OPEX and the financing terms — the exact things underwriting negotiates.

**Base financing assumptions** (econ §3.3, §3.5; all `[A]`, to be struck at warehouse close):

| Parameter | Base | Source |
|---|---:|---|
| Advance rate | **75%** | econ §3.3 (conservative vs Sunrun 79.3%) |
| All-in cost of debt | **~9%** (SOFR + ~350–450 over deal life) | econ §3.1 rung 2/4 |
| Amortization | **5-yr level** (matched to GPU life, NOT solar's 20–25yr) | econ §3.5 |
| NCADS (yr-1) | **$9,870** (§B3, net-of-OPEX-and-reserve) | this part §B3 |

### B4.2 Debt service and DSCR at $20K and $22K CAPEX (base)

```
$20K CAPEX:  loan = 0.75 × 20,000 = $15,000
             5-yr level @ 9% → annual debt service = 15,000 × [0.09 / (1−1.09^−5)]
                                                    = 15,000 × 0.25709 ≈ $3,856/yr
             yr-1 DSCR = NCADS / DS = 9,870 / 3,856 ≈ 2.56×   (using net-of-OPEX)
             yr-1 DSCR on net-to-stream (pre-OPEX) = 12,000 / 3,856 ≈ 3.11×   (≈ Part A "3.2×")

$22K CAPEX:  loan = 0.75 × 22,000 = $16,500
             annual debt service = 16,500 × 0.25709 ≈ $4,242/yr
             yr-1 DSCR (net-of-OPEX)   = 9,870 / 4,242 ≈ 2.33×
             yr-1 DSCR (net-to-stream) = 12,000 / 4,242 ≈ 2.83×   (≈ Part A "3.0×")
```

> Part A's "3.2× / 3.0× base" are **pre-OPEX** DSCRs. **Post-OPEX base DSCR is ~2.6× ($20K) / ~2.3× ($22K)** — still comfortably financeable, but ~0.5× lower than the headline because OPEX + reserve are now honestly subtracted. **The CAPEX move from $20K→$22K costs only ~0.2–0.3× of DSCR** (debt service rises 10%, NCADS unchanged) — confirming CAPEX is a *second-order* DSCR driver.

### B4.3 Stress DSCR (50% util × 60% overlap, price −30% → $0.28; econ §2.3 stress row)

Stress revenue (econ §2.3 stress): gross billings ~$9.8K → net-to-stream ~$6.8K. Apply this part's OPEX: cash OPEX is *lower* in stress (less rented traffic → less relay/cache/electricity), call it ~$950; reserve held at $1,000 (it's a fixed covenant). **Stress NCADS ≈ 6,800 − 950 − 1,000 ≈ $4,850.**

```
$20K stress DSCR (net-of-OPEX)  = 4,850 / 3,856 ≈ 1.26×
$22K stress DSCR (net-of-OPEX)  = 4,850 / 4,242 ≈ 1.14×
   for comparison, Part A's pre-OPEX stress DSCR (6,800/DS): 1.76× / 1.60×  ≈ Part A's "1.8×/1.7×"
```

> **This is the underwriting headline and the most important correction in the doc.** Part A's stress DSCR of ~1.7–1.8× is **pre-OPEX**. **Post-OPEX stress DSCR is ~1.1–1.3×** — still above 1.0× (the asset covers its debt under stress) but the cushion is much thinner than advertised, and **$22K @ stress (~1.14×) sits dangerously close to the econ §3.5 turbo-amortization trigger (3-mo-avg DSCR < 1.5×).** Implication: at $22K CAPEX, a stress scenario *trips the cash trap*. Mitigants underwriting will require: (a) lower advance rate (e.g., 70% → DSCR $20K stress ≈ 1.35×), (b) the contracted fixed legs (HSA + reserved-instance + DR, econ §3.7) lifting the stress floor, (c) the refresh reserve being releasable under a stress waiver. **Carry the post-OPEX stress DSCR into rating-agency pre-reads, not the pre-OPEX number.**

### B4.4 Unlevered IRR (6-yr, post-OPEX, declining stream)

IRR solves NPV = 0 on `[−CAPEX, +net-of-OPEX yr1..6]` from §B3.1 (`$9,870 / 7,590 / 5,790 / 4,270 / 3,140 / 2,200`):

| CAPEX | Pre-OPEX IRR (Part A basis) | **Post-OPEX IRR (this part)** |
|---|---:|---:|
| $20K | ~37% | **~26–28%** |
| $22K | ~33% | **~22–24%** |

> Post-OPEX unlevered IRR lands in the **low-to-mid 20s%** — down from Part A's low-30s/high-30s because OPEX + reserve (~$11.3K cumulative over 6 yr, §B3.1) come out of the cash flows. **Still a strong project-finance return**, and it *levers up* attractively at a 75% advance and ~9% debt (positive leverage: asset yield > cost of debt). The point of restating it: an unlevered IRR of "~22–28% post-everything" is the defensible number; "~45%" (the original deck) and even "~33–37%" (Part A, pre-OPEX) overstate it.

## B5. 60% × 70% utilization SENSITIVITY — the bigger risk than the $4K host

This is the section underwriting cares about most. Part A §A3 already showed the $18K→$22K CAPEX move costs ~0.1–0.3× of DSCR. **A utilization/overlap miss costs multiples of that** — it hits the numerator (NCADS) directly and non-linearly. `platform.md` establishes the regimes; here is the financial translation. This is also risk **R5** in Part D.

### B5.1 Scenario grid (8× 4090, $0.40 base price, 25% fee, $20K CAPEX, 75%/9%/5yr financing)

| Scenario | Util × Overlap | Rented GPU-hr | Net-to-stream | − Cash OPEX | − Reserve | **NCADS** | yr-1 DSCR | Read |
|---|---|---:|---:|---:|---:|---:|---:|---|
| **Commercial high** (econ) | 80% × 70% | 56,064 | ~$15,600 | $1,300 | $1,000 | **$13,300** | **3.45×** | best financeable case |
| **Commercial base ★** | 60% × 70% | 42,048 | $12,000 | $1,130 | $1,000 | **$9,870** | **2.56×** | the underwritten case |
| **Commercial stress** | 50% × 60% | 35,040 | ~$6,800 | $950 | $1,000 | **$4,850** | **1.26×** | near turbo trigger |
| **Residential + space heat** (cold) | ~70% eff. overlap, 12,000 kWh load | ~$11.3K gross | ~$8,500 | $1,000 | $1,000 | **$6,500** | **1.69×** | financeable only seasonal-weighted |
| **Residential DHW-only / warm** | overlap → 30–50% (`platform.md`) | low | ~$3,100 | $700 | $1,000 | **$1,400** | **0.36×** | **NOT financeable** as compute collateral |
| **Warm-climate summer (any)** | overlap → 30–50%, util duty-cycle-capped 65–75% | reduced | — | — | $1,000 | sharply lower | <1.5× | seasonal floor; lean on DR/VPP (`platform.md`) |

> *Residential figures are rough `[A]` and definitional: the ~$3,100 DHW-only net-to-stream here is before the residential owner split; `platform.md` quotes ~$2.3K/unit-yr as the net-to-owner DHW figure (after the ~50% residential owner split, econ §3.2). They are consistent — residential DHW-only does not clear debt service as compute collateral either way.*

### B5.2 How net margin and DSCR move — the elasticity

```
Move from 60%×70% (base) → 50%×60% (stress):
  rented GPU-hr: 42,048 → 35,040  (−17% from util) and price −30% compounds
  NCADS:         $9,870 → $4,850   = −51%   ← net margin HALVES
  DSCR:          2.56×  → 1.26×     = −51%   ← DSCR moves ~1:1 with NCADS

Compare the $4K host (CAPEX $18K→$22K, +22% CAPEX):
  DSCR:          2.56×(@20K base proxy) → 2.33×(@22K)  = −9%

⇒ A 10-point utilization miss + price stress moves DSCR ~5× more than the entire host CAPEX dispute.
```

> **The structural conclusion (consistent with `platform.md` and risk R5).** The 60%×70% assumption is a **commercial annual average**; it is honest for hotels/apartments/campuses with near-continuous draw, but **residential DHW-only and warm-climate/summer regimes fall to 30–50% overlap**, dropping NCADS below the debt-service line (DSCR < 1.0× in the worst row). This is why:
> 1. **Only seasoned, metered commercial units enter financed pools** (`platform.md` tape joins on `seasoned=true`; econ §3.8 R-utilization).
> 2. The tape must be fed **per-region, seasonal-weighted overlap**, not the winter peak (`platform.md`) — or residential pools get over-underwritten.
> 3. The **contracted fixed legs** (HSA min-draw + reserved-instance ≥30% + DR/VPP, econ §3.7) exist precisely to put a floor under NCADS so the *stress* DSCR clears the 1.5× turbo trigger. The DR/VPP $20–100/unit-yr (`platform.md`) is the summer/warm-climate floor.
>
> **OPEX is ~9% of net and barely moves DSCR; utilization × overlap is the whole ballgame.** Spend the underwriting attention there.

## B6. OPEX → tape NET-margin linkage

How the §B2 OPEX feeds the securitization net cash flow that noteholders are paid from.

### B6.1 Where OPEX enters the ledger and the tape

The `securitization_tape` materialized view (`platform.md`) currently surfaces **gross revenue, platform fee, GPU-hours, realized $/GPU-hr, utilization %, reliability tier** — i.e. it stops at **net-to-stream**. It does **not** carry OPEX. For underwriting it must be extended so the tape reports **net-of-OPEX** per node per period:

```
tape NET cash flow (per node, per period)
   = Σ usage_record.gross_usd                         -- metered, double-signed (platform.md)
   − platform_fee (25%)                               -- settlement.platform_fee_usd
   − non-overlap electricity                          -- settlement.electricity_offset_usd (already a column!)
   − allocated cash OPEX (§B2.1 lines 1–10)           -- NEW: per-node OPEX allocation
   − refresh-reserve contribution (§B2.1 line 12)     -- NEW: trapped in SPV before residual
   = node-level NCADS  → aggregates to pool NCADS → SPV waterfall (econ §3.5)
```

`settlement.electricity_offset_usd` already exists (`platform.md`) — line 11 is wired. **The missing fields are a per-node cash-OPEX allocation and the refresh-reserve accrual.** Recommended: add `opex_allocated_usd` and `refresh_reserve_usd` columns to `settlement` and surface their period sums in `securitization_tape`. Allocation method: connectivity/cache/cloud OPEX by **metered GPU-hours** (usage-driven); field/insurance/payout by **per-node flat**; this keeps it reproducible from the append-only `usage_record` table (the audit guarantee, `platform.md`).

### B6.2 The waterfall position (econ §3.5)

```
Pool collections (gross)
  → platform fee out                         (to marketplaces)
  → cash OPEX out (§B2 lines 1–10)           ← NEW explicit line: ~9% of net
  → senior interest + principal (Class A)    ← DSCR measured HERE on net-of-OPEX (§B4)
  → mezz / junior (B / C)
  → refresh reserve top-up (§B2 line 12)     ← trapped; funds yr-5/6 swap (econ §3.5, R4)
  → liquidity reserve top-up                 (6-mo senior interest)
  → RESIDUAL to Superheat                    (the retained annuity, econ §3.5)
```

### B6.3 Why this matters to a rating

An underwriter rating Class A at A/A− (econ §3.5) sizes subordination/OC off **NCADS volatility**, not gross. Three consequences of modeling OPEX honestly on the tape:

1. **The DSCR covenant (§B4) is measured on net-of-OPEX** — so the tape *must* report it or the covenant is unverifiable. A tape that only shows net-to-stream overstates coverage by ~0.5× (§B4.2).
2. **OPEX is low and stable (~9% of net, §B2.5)** — a *positive* for the rating: it means net cash tracks revenue closely and the connectivity line even *improves* with scale (§B2.2). The volatility the rating fears is in revenue (utilization/price), not OPEX.
3. **The refresh reserve is a named credit-enhancement feature** (econ §3.5, the "supplemental GPU-refresh reserve") — surfacing line 12 on the tape proves the reserve is funded from cash flow, which is exactly the novel covenant (R4) that lets this asset clear the data-center-ABS methodologies (econ §3.6). An unfunded refresh reserve is a deal-killer; a tape-visible accrual is the cure.

## B7. Part B recommendations & required actions

1. **Adopt $1,130/unit-yr additive cash OPEX (~$0.027/GPU-hr) + $1,000/unit-yr refresh reserve** as the base OPEX block; replace every `[A]` with metered actuals during Phase A (Part C Phase A metrics already track relay-$/GPU-hr and cache hit-rate — wire them straight into line 1/3).
2. **Restate the underwriting cash flow to net-of-OPEX ($9,870 yr-1, ~18% below the memo's net)** and re-run the waterfall, payback, IRR, and DSCR on it — not on net-to-stream.
3. **Quote payback as a range with the basis stated:** ~22–25 mo (pre-OPEX, declining stream) / ~29–33 mo (post-OPEX-and-reserve). **Never quote the flat-$11.7K ~20–22 mo number** — it is wrong against the declining stream.
4. **Carry the post-OPEX DSCR into rating pre-reads:** base ~2.3–2.6×, **stress ~1.1–1.3×** — and note that **$22K @ stress (~1.14×) trips the 1.5× turbo trigger**, so either lower the advance to ~70% or lean on the contracted fixed legs (HSA/reserved/DR) to lift the stress floor.
5. **Stop debating the $4K host as a financeability question.** §B5 proves a 10-point utilization/overlap miss moves DSCR ~5× more than the entire host CAPEX dispute. Put underwriting attention on **per-region seasonal-weighted overlap** (`platform.md`) and the **contracted-revenue ratio** (econ §3.7).
6. **Extend the tape** (`platform.md`): add `opex_allocated_usd` and `refresh_reserve_usd` to `settlement` + `securitization_tape`, so DSCR is measurable on net-of-OPEX and the refresh reserve is provably funded (§B6).

---

# Part C — Engineering Build Roadmap

This part expands the build-phasing and open-questions sections of `system-design-overview.md` into a concrete, executable engineering roadmap, and ties every technical deliverable to the phased financing ladder in `../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md` (§3.1, §3.9) and the securitization mechanics in `../market-research/Superheat_Compute-Network_Debt-Engine_Onepager.md`.

The governing thesis is unchanged: **the metered revenue ledger is the product.** Engineering does not just keep GPUs rented; it manufactures a clean, auditable, per-unit securitization tape. Every phase below is sequenced so that the *technical* exit criteria are exactly the *financial* entry criteria for the next rung of the funding ladder (asset-proving → warehouse → inaugural ABS → programmatic).

This part cross-references the sibling design docs that spin out of `system-design-overview.md`: `node.md`, `connectivity-and-data.md`, `platform.md`, `security-and-compliance.md`, `operations.md`, and `data-contracts.md`.

## C1. Roadmap principles (how phasing is decided)

1. **Revenue before margin before moat.** Phase A attaches to existing marketplaces for the fastest path to a metered cash flow; Phase B captures the margin that fee-taking marketplaces skim; Phase C owns the demand and the UX. This is the overview's build sequence, made concrete.
2. **The tape gates the capital, the capital gates the build.** No phase is "done" because the code shipped. It is done when it has produced the *financial artifact* the next funding rung underwrites (12 months of clean per-unit tape; a seasoned ≥70%-utilized warehouse pool; a ratable ABS collateral package).
3. **Fail-safe and outbound-only are not phases — they are invariants.** The engineering principles in `system-design-overview.md` hold from the first node. We never ship a Phase-B feature that can brick a Phase-A unit or open an inbound port.
4. **Build the moat, buy the boring parts.** We buy managed Postgres, time-series storage, and fleet/OTA tooling; we build the scheduler, thermal coordinator, yield router, and metering ledger.
5. **Every deliverable carries a financing tag.** Each node and each GPU-hour is tagged with its financing tranche/SPV from day one (registry in `platform.md`), because the warehouse and ABS need per-unit, per-tranche segregation of collateral.

## C2. Phase A — "Attach for revenue"

**One-line outcome:** nodes earn on marketplaces with safe remote ops, producing the metered tape that Phase-1 financing (equity-funded pilot + venture debt, ladder rungs 0–1) underwrites.

### C2.1 Scope / deliverables
- **Node agent v1** (`node.md`): signed control daemon — heartbeat, telemetry (power, GPU temp/util, tank temp, heat demand, uptime, network quality), container lifecycle, OTA orchestration, local hardware watchdog.
- **Immutable OS with A/B updates + automatic rollback** (`node.md`): atomic Linux base (Ubuntu Core / Flatcar / Talos-style), NVIDIA driver + Container Toolkit baked in, read-only root, signed images only.
- **WireGuard outbound-only overlay + relay/TURN fallback** (`connectivity-and-data.md`): every node dials out; zero inbound ports; CGNAT relay path measured and budgeted.
- **Thermal guard (local safety only)** (`node.md`): reads tank temp / heat demand, enforces hard local power-state limits. *Local safety always wins over remote scheduling.* This is the guard, not yet the optimizing coordinator (that is Phase B).
- **Telemetry pipeline + minimal registry** (`platform.md`): per-node inventory, SKU, region, owner, financing tag; time-series ingest; basic dashboards/alerts.
- **vast.ai host attach connector** (`platform.md`): nodes accept marketplace jobs through the agent's isolated containers; this is the fastest path to revenue.
- **Metering v0 — read-only tape** (`platform.md`): capture per-unit GPU-hours delivered and realized $/hr *as reported by the marketplace*, reconciled against telemetry. Authoritative billing comes in Phase B; Phase A proves the tape is *capturable and auditable*.

### C2.2 Implements which sibling docs
`node.md` (OS + agent + thermal guard, SKU freeze + thermal interface), `connectivity-and-data.md` (overlay + relay), `platform.md` (registry, telemetry, OTA, metering v0, vast.ai connector + concentration logging), `security-and-compliance.md` (tenant isolation, egress firewall, software attestation baseline), `operations.md` (telemetry pipeline, dashboards/alerts).

### C2.3 Entry criteria
- Commercial SKU **frozen** (resolves Risk R3 — 8 vs 12 GPU) so BOM/thermal/tape definitions are stable.
- ≥1 pilot site with controllable thermal load (commercial host preferred — near-continuous draw, the financeable class per Part B §B1).
- Equity pilot capital committed (ladder rung 0, ~$2–5M).

### C2.4 Exit criteria (= Phase-1 financing entry)
- **≥1,000 commercial units** deployed and instrumented (economics memo §3.9 Phase 1).
- **12 consecutive months of clean per-unit monthly tape**: metered GPU-hours, realized $/hr, utilization, uptime, thermal draw, host payment behavior.
- A/B rollback proven in production (canary → cohort → fleet) with zero customer-hot-water incidents.
- Marketplace-concentration metric instrumented and reported (no single platform >60% — the trigger in `platform.md` and economics memo §3.8).

### C2.5 Key metrics
Per-unit metered GPU-hr/mo · realized $/GPU-hr · utilization % · node uptime % · thermal availability % · OTA success/rollback rate · relay-bytes vs direct-bytes ratio (cost) · % revenue per marketplace (concentration).

### C2.6 Team / skills
Embedded Linux / immutable-OS engineer; fleet/OTA + reliability engineer; networking engineer (WireGuard/NAT/CGNAT); firmware-adjacent thermal-controls engineer; data engineer (telemetry + tape); 1 backend engineer (registry/connector). ~5–7 engineers.

### C2.7 Maps to financing ladder
**Rung 0–1: equity-funded pilot fleet → venture debt / equipment lines** (economics memo §3.1). The deliverable Phase A owes finance is the **legible asset**: a credit file an underwriter can read (`Superheat_Compute-Network_Debt-Engine_Onepager.md` §8 Phase 1). This is "asset-proving."

## C3. Phase B — "Own the orchestration"

**One-line outcome:** Superheat captures the marketplace fee and establishes a real utilization floor, turning the tape from "reported by others" into an **authoritative, owned metering ledger** — the securitization tape proper. Unlocks the **warehouse facility** (ladder rung 2).

### C3.1 Scope / deliverables
- **Control-plane scheduler** (`platform.md`): matches demand to nodes subject to thermal availability + network quality + reliability score + interruptibility.
- **Thermal coordinator** (`platform.md`): per-site tank-as-battery model — current temp, heat-demand forecast, comfort constraints, storage headroom; issues run-windows and pre-heat schedules. Hard comfort/safety constraints inviolable.
- **Metering + billing ledger (the securitization tape)** (`platform.md`): authoritative per-unit GPU-hours, $/GPU-hr realized, owner settlement, platform take. Accurate, auditable, per-unit, per-tranche from day one — this *is* the tape of `Superheat_Compute-Network_Debt-Engine_Onepager.md` and economics memo §3.5.
- **Interruptible batch / inference backlog filler** (`platform.md`): Superheat-owned/partner preemptible work so a thermal window is never idle.
- **Reliability scoring + tiering** (`platform.md`): continuous per-node score (uptime, thermal availability, network quality) → high-reliability nodes get premium/contracted work; the rest get spot/filler. Produces the seasoned, high-reliability sub-pools financing wants.
- **Yield router v1** (`platform.md`): per node-hour, pick highest expected $/GPU-hr across available sources, honoring thermal window, interruptibility, reliability tier, and the **60% concentration guard**.
- **Checkpoint/resume v1** (`connectivity-and-data.md`): mandatory for the interruptible filler to be safe — checkpoint to local NVMe, resume elsewhere.
- **DR/VPP enrollment (begin)** (`platform.md`): start enrolling flexible load (EnergyHub/Renew Home per partnership map) to stack a fixed contracted leg.

### C3.2 Implements which sibling docs
`platform.md` (scheduler, thermal coordinator, reliability scoring, authoritative ledger, thermal battery + pre-heat, interruptible filler, reliability tiering, yield router + backlog), `connectivity-and-data.md` (checkpoint/resume v1).

### C3.3 Entry criteria
- Phase A exit met (12-month tape on ≥1,000 units).
- Thermal-demand seasonality model started per region (Risk R5) so utilization promises to lenders are honest.
- Pre-engagement with a warehouse lender / rating agency begun on the Phase-A tape (economics memo §3.6, §3.9 Phase 2).

### C3.4 Exit criteria (= warehouse-facility entry)
- **Authoritative owned ledger reconciles to within tolerance** against marketplace settlement and bank cash receipts (tape integrity — Risk R8).
- **Trailing-6-month DSCR ≥ 2.5×** on the candidate pool (economics memo §3.9 Phase 2 exit).
- **≥30% of fleet GPU-hours reserved/contracted** (reserved-instance + HSA + DR), per economics memo §3.9 Phase 1/§3.7.
- Warehouse **≥70% utilized** against the seasoned fleet; rating-agency pre-read complete.
- Backup-servicer runbook/code escrow drafted (Risk R7) — the standard esoteric-ABS cure raised early.

### C3.5 Key metrics
Marketplace-fee captured (vs paid) · utilization floor lift from filler (Δ idle-hours) · ledger reconciliation variance · DSCR (trailing-6-mo) · % contracted revenue · reliability-score distribution · checkpoint success rate / mean preemption recovery time · DR/VPP $/unit-yr enrolled.

### C3.6 Team / skills
Distributed-systems engineers (scheduler, yield router); optimization/controls engineer (thermal coordinator + forecasting); data/ledger engineer + a finance-systems engineer for billing reconciliation and tape audit; SRE for fleet at thousands of nodes; ML/infra for backlog filler. ~8–12 engineers, plus finance-ops liaison.

### C3.7 Maps to financing ladder
**Rung 2: warehouse SPV facility** ($25–50M revolving, SOFR+250–450, ~70–75% advance — economics memo §3.1/§3.9 Phase 2). Engineering owes finance an **owned, auditable tape + seasoned high-reliability pool + ≥30% contracted revenue**. This is "warehouse."

## C4. Phase C — "Own the demand & UX"

**One-line outcome:** full hybrid yield routing and a smooth renter experience — Superheat owns its demand channel and UX, maximizing realized $/GPU-hr and contracted-revenue ratio. Unlocks the **inaugural term ABS** (rung 4) and **programmatic issuance** (rung 5).

### C4.1 Scope / deliverables
- **Own marketplace / direct enterprise API** (`platform.md`): higher-margin demand that avoids the 25–35% marketplace fee.
- **Reserved-instance contracts** (`platform.md`): contracted floors that convert spot exposure to contracted exposure — direct rating fuel (economics memo §3.7).
- **Regional content-addressed cache + pre-staging** (`connectivity-and-data.md`): popular base images, weights, datasets served from regional PoPs + on-node cache, so jobs don't pull over slow home uplinks; dedup across the fleet.
- **Checkpoint/resume at scale + bandwidth-aware placement** (`connectivity-and-data.md`): place data-heavy jobs on better-connected nodes; route bulk transfers over direct mesh paths, not paid relays.
- **VPP / demand-response at fleet scale** (`platform.md`): full enrollment for hours when neither compute nor heat is wanted.
- **Trusted-host tier** (`security-and-compliance.md`): vetted commercial / TEE-capable sites for sensitive workloads, priced and tracked separately (sizes the addressable-market limit of Risk R1).
- **Smooth renter entry** (`connectivity-and-data.md`, `platform.md`): one-command container/SSH via the overlay; persistent volumes with clear eviction semantics.

### C4.2 Implements which sibling docs
`platform.md` (own marketplace/API, reserved contracts, full yield routing, fleet-wide VPP/DR, reserved floors), `connectivity-and-data.md` (regional cache, checkpoint/resume at scale, bandwidth-aware placement, renter UX), `security-and-compliance.md` (trusted-host tier).

### C4.3 Entry criteria
- Warehouse drawn and performing (rung 2 live); inaugural-ABS structuring engaged with bookrunner + rating agencies (economics memo §3.5, §3.9 Phase 3).
- Owned-demand pipeline (enterprise offtake / reserved interest) qualified.

### C4.4 Exit criteria (= inaugural ABS → programmatic)
- **≥10,000 seasoned units (≥12-mo metered history)** available as collateral (economics memo §3.5 pool).
- **Inaugural ABS prices inside SOFR+300 blended at ≥75% advance** (economics memo §3.9 Phase 3 exit).
- Contracted-revenue share **rising over the deal's life** to satisfy the collateral covenant (economics memo §3.5).
- Servicer-replacement + backup-servicer fully executed and tested (Risk R7) — required to clear the rating committee.
- Cache hit-rate and relay-cost reduction demonstrated (controls Risk R2 opex line).

### C4.5 Key metrics
Realized $/GPU-hr on own-demand vs marketplace · contracted-revenue ratio (trend) · cache hit rate / relay $ per GPU-hr · renter time-to-first-job · trusted-host tier revenue & utilization · ABS advance rate / blended spread.

### C4.6 Team / skills
Product + frontend (marketplace/API, renter UX); enterprise sales-engineering; storage/CDN engineers (regional cache, content-addressing); network-placement engineer; security engineer (trusted-host tier, attestation); structured-finance-systems engineer (covenant reporting, static-pool curves per economics memo §3.8). Scales to ~15–25 engineers across squads.

### C4.7 Maps to financing ladder
**Rung 3–5: forward flow (residential) → term ABS takeout → programmatic issuance + IG path** (economics memo §3.1/§3.9 Phases 3–4). Engineering owes finance a **ratable collateral package**: seasoned diversified pool, rising contracted share, executed backup-servicer, and covenant-grade reporting. This is "inaugural ABS → programmatic."

## C5. Cross-cutting milestone table — engineering ↔ financing

| Eng phase | Key engineering deliverables | Financial artifact produced | Funding ladder rung (economics §3.1) | Financing milestone (economics §3.9 / onepager §8) | Timeline |
|---|---|---|---|---|---|
| **A — Attach for revenue** | Node agent v1, immutable OS + A/B rollback, WireGuard overlay + relay, thermal guard, telemetry/registry, vast.ai attach, metering v0 | Clean 12-mo per-unit metered tape on ≥1,000 commercial units; concentration metric | Rung 0–1 (equity pilot → venture debt/equipment lines) | **Phase 1 — Tape building / asset-proving** | now → 12 mo |
| **B — Own orchestration** | Scheduler, thermal coordinator, **authoritative metering ledger (the tape)**, interruptible filler, reliability scoring/tiering, yield router v1, checkpoint/resume v1, begin DR/VPP | Owned auditable tape; seasoned high-reliability sub-pool; DSCR ≥2.5× trailing-6-mo; ≥30% contracted revenue | Rung 2 (warehouse SPV facility) | **Phase 2 — Warehouse + forward flow** | 12 → 24 mo |
| **C — Own demand & UX** | Own marketplace/API, reserved contracts, regional cache + pre-staging, checkpoint/resume at scale, bandwidth-aware placement, fleet VPP/DR, trusted-host tier, renter UX | Ratable diversified pool (≥10k seasoned units); rising contracted share; executed backup-servicer; covenant reporting | Rung 3–4 (forward flow → term ABS takeout) | **Phase 3 — Inaugural term ABS (Compute & Thermal Trust 2027-1)** | 24 → 36 mo |
| **C+ — Scale/operate** | Multi-region operation, GPU-refresh tooling (year-5 modular swap), programmatic covenant automation | Programmatic-grade collateral; refresh-reserve execution capability | Rung 5 (programmatic + IG path) | **Phase 4 — Programmatic** | 36 mo+ |

---

# Part D — Consolidated Risk Register

This pulls together every open question and risk distributed across the design docs and the market-research risk sections (economics memo §3.8, onepager §7, partnership map §8) into one engineering-owned register. **Likelihood** and **impact** are High/Med/Low. **Owner-area** names the design doc / team that owns the mitigation. The final column states the concrete decision or experiment that *resolves* (closes or de-risks) the item.

## D1. The register (R1–R15)

| # | Risk | Impact | Likelihood | Owner-area | Mitigation | Resolving decision / experiment |
|---|---|---|---|---|---|---|
| **R1** | **Confidential compute / TEE limits on 4090.** Consumer 4090s lack a robust hardware TEE, so we cannot promise hardware-enforced confidentiality vs a determined host on standard nodes. Caps addressable market to non-sensitive workloads. | High (TAM ceiling) | High (hardware fact, certain) | `security-and-compliance.md` | Ephemeral + encrypted-at-rest, no-persistence by default (wipe on job end); software attestation of image+agent; separate **trusted-host tier** (vetted/TEE-capable commercial sites) priced higher and tracked separately; be explicit to renters about tier. | **Experiment:** measure addressable demand that *requires* hardware TEE vs not; size trusted-host tier revenue (Phase C). **Decision:** set % of fleet that must be TEE-capable hardware; track 4090-successor / data-center-GPU TEE roadmaps for the refresh cycle. |
| **R2** | **CGNAT relay cost + slow upload.** P2P hole-punching fails behind CGNAT → traffic falls to paid relays; asymmetric consumer uplinks throttle result/checkpoint egress. Adds opex and caps data-heavy workloads. | Med–High (opex + workload mix) | High (most residential links) | `connectivity-and-data.md` | Prefer direct mesh paths; regional content-addressed cache + on-node cache so jobs pull weights locally; bandwidth-aware placement of data-heavy jobs; route bulk over direct paths; budget relay as an explicit cost line; consider cellular/secondary link for commercial sites. | **Experiment:** instrument relay-bytes vs direct-bytes and $/GPU-hr relay cost in Phase A; pilot regional cache in Phase C and measure hit-rate + relay-$ reduction. **Decision:** per-region cache placement; relay budget cap feeding the underwriting opex line (Part B §B2.2). |
| **R3** | **Commercial SKU definition — 8 vs 12 GPU.** Pitch says 12 GPU / 33 kW; economics memo underwrites 8× 4090 ≈ 3.3 kW. Drives BOM, thermal design, and the underwriting tape; conflict must be resolved before hardware freeze. | High (BOM + tape consistency) | High (open today) | `node.md` | Standardize commercial SKU at **8× 4090** to match underwritten economics (Part A §A1); treat 12-GPU/33 kW as optional larger config; *or* re-underwrite at 12. One definition must drive BOM, thermal, and tape. | **Decision (hardware freeze, Phase A entry):** pick 8× 4090 as the standard underwritten SKU, or formally re-underwrite at 12 with new thermal + economics. Blocks Phase A start. |
| **R4** | **Year-5 modular GPU-refresh.** Securitization funds a year-5 module swap (refresh reserve); if hardware/software isn't designed for modular replacement + version-aware re-registration, the second revenue cycle and a key ABS covenant fail. | High (covenant + 2nd-cycle revenue) | Med | `node.md`, `platform.md` (registry) | Design node for modular GPU replacement; version-aware re-registration in the control-plane registry; consider leasing compute modules (Dell/HPE FS per partnership map §4) to shift CAPEX off the SPV and simplify the refresh covenant. | **Experiment:** field-test a GPU module swap + re-registration on a pilot unit. **Decision:** modular-replacement requirement in the hardware spec; refresh-reserve mechanics + lease-vs-own confirmed before ABS close (Part B §B2.4). |
| **R5** | **Heat-demand seasonality affecting utilization.** Utilization assumptions rest on thermal overlap (70% commercial). Cold vs warm climate and summer DHW-only periods change the achievable rented hours → over-promised utilization to financiers. | High (utilization promises) | High (physical/seasonal) | `platform.md` (thermal coordinator) | Tank-as-thermal-battery + pre-heat to decouple compute from instantaneous heat demand; interruptible backlog filler so windows aren't wasted; per-region thermal-demand modeling; commercial-first (near-continuous draw); paid-electricity hours kept margin-positive when rented. | **Experiment:** per-region seasonal thermal-overlap study using Phase A telemetry. **Decision:** region-adjusted utilization curves fed into underwriting (no single flat 70% assumption); HSA minimum-draw covenants on commercial hosts. See Part B §B5. |
| **R6** | **Marketplace concentration / 60% trigger.** One marketplace's fee or policy change hits the whole pool; >60% single-platform revenue is an ABS cash-trap trigger. | High (revenue + ABS trigger) | Med | `platform.md` | Yield router enforces a concentration guard; multi-home across vast.ai, RunPod, Salad, io.net, Akash; build own demand + reserved/enterprise contracts (Phase C) to dilute marketplace share. | **Experiment:** track per-marketplace revenue share continuously from Phase A. **Decision:** hard covenant ≤60% per platform; Phase-C own-demand ramp target to keep any single platform under trigger. |
| **R7** | **Servicer concentration / backup servicer.** "Only Superheat can run the fleet" is a concentration risk that blocks the rating committee. | High (blocks ABS) | Med | `security-and-compliance.md`, `platform.md` | Backup-servicer agreement + escrowed orchestration runbooks/code (standard esoteric-ABS cure); make orchestration auditable; servicer-replacement triggers with a named backup servicer in the deal. | **Decision/experiment:** draft escrow + backup-servicer runbook in Phase B; **test a servicer-failover drill** before inaugural ABS (Phase C exit). |
| **R8** | **Securitization-tape integrity.** The metering/billing ledger *is* the collateral tape; inaccuracy, gaps, or non-reconciliation between telemetry, marketplace settlement, and bank receipts destroys the deal's credibility (the gain-on-sale / Carvana credibility trap). | Critical (kills deals) | Med | `platform.md` | Authoritative per-unit, per-tranche ledger from day one; reconcile telemetry ↔ marketplace settlement ↔ bank cash; immutable/audited records; publish static-pool curves quarterly; retain residual (skin in the game, Reg-RR). | **Experiment:** monthly three-way reconciliation variance report starting Phase B; external audit of the tape before warehouse close. **Decision:** reconciliation tolerance threshold as a Phase-B exit gate. |
| **R9** *(cross-cutting)* | **Fail-safe-to-heater regression.** Any OS/agent update, crash, or compute fault that leaves a customer without hot water violates the core invariant and threatens the whole deployment relationship. | Critical (per-customer + brand) | Low (with controls) | `node.md` | A/B updates + automatic rollback; hardware watchdog; recovery ladder ending in "operate as ordinary water heater, compute disabled"; staged OTA (canary→cohort→fleet) with auto-halt. | **Experiment:** chaos/fault-injection on the recovery ladder in Phase A; zero-hot-water-incident requirement as a standing release gate. |
| **R10** *(cross-cutting)* | **Issuance-market freeze.** The flywheel assumes structured-credit markets keep buying; a freeze forces hardware onto Superheat's balance sheet (the 2008 / Mosaic failure mode). | High (liquidity) | Low–Med | Finance (eng supports via committed-pipeline reporting) | Committed multi-year warehouse + forward-flow capacity ≥12 months of origination; growth governed by *committed* capital, never assumed take-out; hardware margin + servicing + compute are independent legs. | **Decision (finance):** never let deployment run-rate exceed committed facility capacity; engineering provides the committed-vs-deployed dashboard. |
| **R11** | **Regulatory / electrical-code / permitting / UL certification.** Installing an 8×4090/~3.3–4 kW (or H1) GPU appliance in homes & commercial buildings requires NEC-compliant circuits, permits + inspection per site, and UL/ETL listing of the combined heater+GPU appliance (plus NSF/plumbing for potable water). A failed listing or a jurisdiction that won't permit it blocks deployment there. | High | Med | `security-and-compliance.md`, `node.md` | Certify the appliance early; installer pulls permits; per-region compliance gating checklist before entry. | **Resolving:** obtain UL/ETL listing + a reference permit in the pilot jurisdiction before scaling (Phase A). |
| **R12** | **Homeowner churn / node recovery (collateral loss).** The collateral lives in third parties' homes; a homeowner who sells, unplugs, or revokes the unit removes a financed asset from the pool — the esoteric-ABS analog to auto-loan repossession, with no easy repo. | High (collateral integrity + tape) | Med | `operations.md`, `platform.md` (registry/tape) | Host agreement with term + transfer-on-sale clause; node-dark vs node-removed detection; collateral write-down/substitution rules in the tape; re-siting/redeployment of recovered units; over-collateralization absorbs expected churn. | **Resolving:** measure real churn/removal rate in Phase A; set a pool churn assumption + substitution mechanic before warehouse close. |
| **R13** | **Insurance / product liability.** A GPU array thermally coupled to a pressurized potable-water tank in someone's home: fire, electrical, water-damage, and product-liability exposure. | High | Low–Med | `security-and-compliance.md` | Product-liability + commercial policy; clear homeowner-policy vs Superheat-policy boundaries; installer liability; warranty; the fail-safe-to-heater design as a liability-reducing claim. | **Resolving:** bind product-liability coverage + define the policy stack before scaled deployment. |
| **R14** | **Data privacy / compliance.** Renter data resides/transits on third-party residential premises, and the thermal forecaster inevitably learns homeowner occupancy/presence patterns — both GDPR/CCPA-relevant; breach exposure on consumer hardware (ties to the no-TEE limit). | Med–High | Med | `security-and-compliance.md` | Data-minimization on occupancy signals; encryption/wipe (with the honest live-memory caveat); breach-notification process; data-residency per region; renter ToS + homeowner consent. | **Resolving:** privacy review + DPIA before EU/CCPA-state entry. |
| **R15** | **Electricity tariff / net-metering / TOU risk.** The economics assume heat-offset makes compute electricity near-free; changes to residential tariffs, time-of-use rates, or net-metering shift margin, especially for non-overlap (paid-electricity) rented hours. | Med | Med | Part B (financial model), `platform.md` | Keep paid-electricity rented hours margin-positive (yield router floor); per-region tariff modeling; bias non-overlap hours to VPP/DR; pass-through where possible. | **Resolving:** per-region tariff sensitivity in the financial model (Part B §B5); tariff-change monitoring. |

## D2. How the register feeds the build

Three risks are **start-blocking** and must resolve at or before Phase A entry: **R3** (SKU freeze — blocks BOM/thermal/tape; see Part A), **R9** (fail-safe recovery ladder — invariant), and the instrumentation needed to even *observe* R2/R5/R6/R8 (telemetry + per-unit tape). Phase A is therefore as much a *measurement* program as a revenue program: it exists to convert the open questions of the overview into quantified, underwritable parameters.

Phase B closes the financially load-bearing risks: **R8** (tape integrity, via three-way reconciliation), **R5/R6** (utilization seasonality and concentration, via the thermal coordinator, filler, and yield-router guard), and it *drafts* **R7** (backup servicer) early because the rating committee will ask.

Phase C resolves the deal-clearing risks: **R7** (executed/tested backup servicer), **R1** (trusted-host tier sizing), **R2** (cache-driven relay-cost reduction), and stands up **R4** (modular refresh tooling) ahead of the year-5 covenant.

The newer items (**R11–R15**) are predominantly Phase-A/regulatory-gating risks (R11 certification/permitting, R13 insurance, R14 privacy — each gates market entry) and Phase-C scale risks (R12 homeowner churn and R15 tariff exposure, which bind as the pool grows into residential and across regions).

The result is the intended through-line of `system-design-overview.md`: a fleet of unreliable, NAT'd, heat-constrained single-GPU boxes turned into one standardized, near-zero-idle compute fleet **with a metered revenue ledger clean enough to securitize** — built in the exact order the capital needs it.

---

*Cross-references:* `system-design-overview.md` (build phasing + open questions), `node.md` (8-vs-12 SKU freeze, BOM reconciliation, modular-refresh hardware, OS + agent + thermal guard), `connectivity-and-data.md` (relay/cellular/cache cost — OPEX lines 1–3; CGNAT relay; checkpoint/resume; regional cache; renter UX), `platform.md` (metering ledger → securitization tape, cloud/TSDB/Postgres/payout — OPEX lines 4–7, tape extension for net-of-OPEX, scheduler/thermal-coordinator/yield-router/reliability-tiering, the 60%×70% revenue assumptions and seasonal overlap), `security-and-compliance.md` (TEE/trusted-host tier, backup-servicer, UL/permitting, insurance/liability, data privacy), `operations.md` (monitoring/observability/on-call — OPEX line 8, field-service/RMA — OPEX line 9, homeowner churn/node recovery), `data-contracts.md` (schema definitions). *Source:* `../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md` (revenue stream §2.4, financing ladder §3.1/§3.9, waterfall §3.5, DSCR §3.3, risks §3.8), `../market-research/Superheat_Compute-Network_Debt-Engine_Onepager.md` (§7, §8), `../market-research/Superheat_Partnership_and_Capital_Map.md` (§4, §6, §8).

*All `[A]`-marked figures are assumptions to be replaced with metered actuals as the Phase-A tape accumulates. Financing structures are proposed, not executed; validate with structuring counsel and an ABS bookrunner.*
