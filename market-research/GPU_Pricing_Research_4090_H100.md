# GPU Cloud Pricing Research — RTX 4090 & H100 Hourly Rates

**Prepared:** June 12, 2026 · **Scope:** Hourly cloud/marketplace rental rates for NVIDIA H100 and RTX 4090, the historical price trajectory 2023–2026, and what shape the decline curve actually takes.
**Bottom line up front:** The decline is **not linear**. Observed pricing follows a three-phase curve — (1) a steep oversupply crash of −60 to −75% over ~18–24 months, (2) **exponential decay toward a marginal-cost floor** at roughly −25 to −30%/year for mature cards, then (3) noisy stabilization around that floor, including outright price *increases* in 2026. A constant-percentage (geometric) decline with a hard floor fits the data; a straight line does not.

---

## 1. H100 — Hourly Rental Price History (2023 → 2026)

The cleanest longitudinal dataset is Silicon Data's market analysis, which separates **hyperscalers** (AWS/GCP/Azure) from **marketplaces** (Vast.ai-style) and **neoclouds** (Lambda, CoreWeave-style). The tiering matters: hyperscaler list prices run 3–6× the lowest neocloud/marketplace rates for the same silicon.

| Period | Hyperscaler (median $/GPU-hr) | Marketplace | Neocloud | Notes |
|---|---|---|---|---|
| 2023 H2 (Aug–Dec) | $7.76 | — | — | Scarcity era; only hyperscalers, tightly clustered ~$7.7 |
| 2024 Q1 | $7.92 (range $7.37–8.24) | — | — | Peak scarcity; monthly highs in low $8s |
| 2024 H2 | $9.34 | $2.58 | $2.99 | Marketplaces enter Jun 2024, neoclouds Jul 2024 — instant 3×+ tiering |
| 2025 Q1 | $8.96 | $2.29 | $3.50 | Marketplace keeps grinding down |
| **Jun 2025** | **$6.94** | **$2.00** | $3.29 | **Inflection: AWS cuts H100 ~30–45%, triggering a market-wide reset** |
| 2025 H2 | $6.26 | $1.95 | $3.33 | Post-reset stabilization at lower tier levels |
| Dec 2025 – Jan 2026 | — | $2.00 → **$2.20 (+10%)** | — | Year-end training deadlines + budget cycles; isolated to H100 (A100/B200 flat) |
| Mar 2026 | — | 1-yr contract rates $1.70 → $2.35 | — | Modest recovery off the late-2025 lows |
| May–Jun 2026 | AWS p5 ~$3.90–7.50, GCP ~$3.00, Azure $3.40–6.98 | Vast.ai $1.87 → **$2.58** | Lambda $2.86–2.99, CoreWeave $2.50–3.11 | NVIDIA reportedly raised H100 rental pricing ~20%; market median $2.29–3.12 |

Key magnitudes:

- **Peak-to-trough crash:** ~$8/hr (early 2024) → ~$1.50–2.00/hr (late 2025) = **−64 to −75% in under two years** (≈ −50%/yr geometric during the crash phase). ([CloudZero](https://www.cloudzero.com/blog/h100-gpu-cost/), [Silicon Data](https://www.silicondata.com/blog/h100-rental-price-over-time))
- **Mature-phase trend:** marketplace median $3.19 (Jun 2024) → $1.95 (late 2025) = −39% over ~18 months ≈ **−28%/yr geometric**. ([Silicon Data](https://www.silicondata.com/blog/h100-rental-price-over-time))
- **2026 reversal:** prices have *risen* off the floor — the +10% Dec–Jan spike ([Silicon Data](https://www.silicondata.com/blog/h100-price-spike)), NVIDIA's ~20% rental-price increase to roughly $3.60/hr averages ([Crypto Briefing](https://cryptobriefing.com/nvidia-h100-rental-price-increase-2026/)), and May–Jun 2026 escalations at Vast.ai ($1.93→$2.58) and Verda ($2.29→$3.25) ([Thunder Compute](https://www.thundercompute.com/blog/ai-gpu-rental-market-trends)).
- **Current spread (Jun 2026):** $1.38/hr (Thunder Compute) to ~$11–14.90/hr (Oracle bare-metal, premium Azure SKUs) — a 10–23× spread across providers for the same GPU. Median across 42 providers / 247 listings: **~$2.95–3.12/GPU-hr**. ([IntuitionLabs](https://intuitionlabs.ai/articles/h100-rental-prices-cloud-comparison), [GetDeploying](https://getdeploying.com/gpus/nvidia-h100), [AIMultiple](https://aimultiple.com/gpu-index))

Why the crash happened (per Silicon Data): NVIDIA's 2024 supply ramp, marketplace price discovery entering a previously rigid market, the demand shift from training to inference, software efficiency gains cutting GPU-hours per job, and providers having overestimated 2023-era training demand.

### H100 hardware (capex) side — same curve, steeper

- New H100 SXM5: **$35,000–40,000** (late 2023) → still $35–40K list new, but **secondary market $6,000–15,000** by May 2026 = −60 to −85% asset-value decline in ~2.5 years. ([CloudZero](https://www.cloudzero.com/blog/h100-gpu-cost/))
- Driver: generation succession — running inference on an H100 costs ~11× more than on a B300 per unit of work, so older silicon reprices toward its operating-cost floor.

---

## 2. RTX 4090 — Hourly Rental Price History

The 4090 (launched Oct 2022, $1,599 MSRP) tells a very different story from the H100: **it never had a hyperscaler scarcity premium to crash from**. Consumer cards entered the rental market cheap, via marketplaces, and have been **comparatively stable near their floor** the whole time.

| Snapshot | Rate ($/GPU-hr) | Source |
|---|---|---|
| RunPod Secure Cloud, through Apr 2026 | $0.59 (then delisted) | [SynpixCloud](https://www.synpixcloud.com/blog/rtx-4090-cloud-rental-worth-it) |
| RunPod Community Cloud, Apr–Jun 2026 | $0.34 | [RunPod](https://www.runpod.io/gpu-models/rtx-4090), [SynpixCloud](https://www.synpixcloud.com/blog/rtx-4090-cloud-rental-worth-it) |
| Vast.ai marketplace, Jun 2026 | $0.38–0.41 listed; lows ~$0.17 tracked | [Vast.ai](https://vast.ai/pricing/gpu/RTX-4090), [GetDeploying](https://getdeploying.com/gpus/nvidia-rtx-4090) |
| Salad (distributed/consumer hosts), Jun 2026 | $0.18 | [GetDeploying](https://getdeploying.com/gpus/nvidia-rtx-4090) |
| Median, 315 listings / 13 providers, Jun 12 2026 | **$0.54 on-demand · $0.42 spot · $0.49 reserved** | [GetDeploying](https://getdeploying.com/gpus/nvidia-rtx-4090) |
| Index median, 58 providers, May 2026 | **$0.44** | [AIMultiple](https://aimultiple.com/gpu-index) |
| Full market range, 2026 | $0.08–1.61 | [GetDeploying](https://getdeploying.com/gpus/nvidia-rtx-4090), [gpus.io](https://gpus.io/en/gpus/rtx4090) |

Trend observations:

- GetDeploying's own trend note: 4090 on-demand "has remained **stable**, averaging around $0.63/hr" over its tracking window, with the current median at $0.54 — i.e., low-double-digit annual drift, not a crash. ([GetDeploying](https://getdeploying.com/gpus/nvidia-rtx-4090))
- AIMultiple's index (monthly snapshots Jul 2024 → May 2026): "mainstream cards remained relatively stable" while newest-generation GPUs (B200/B300) roughly **doubled** over the past year. ([AIMultiple](https://aimultiple.com/gpu-index))
- The 4090 is now the **cheapest training-class card per hour** ($0.44 median), and its successor exists at only a modest premium — RTX 5090 median **$0.69/hr** — implying the per-generation rental step-down for consumer cards is ~35–40%, absorbed over a 2-year cadence ≈ **−20%/yr at constant compute tier**. ([AIMultiple](https://aimultiple.com/gpu-index))
- Floor evidence: the lowest sustained 4090 listings ($0.15–0.18/hr spot at Salad/Vast) sit just above a host's marginal cost. A 4090 at ~450W draws ~0.45–0.55 kWh per GPU-hour including system overhead; at $0.10–0.15/kWh residential power that is **$0.05–0.08/hr of electricity alone**, before bandwidth, wear, and host margin — so ~$0.15/hr is approximately the rational supply floor, and the market has found it.

---

## 3. Comparison Cards — the Depreciation Precedent

| GPU (launch) | Peak/early rental | Mid-2024 | Mid-2026 | Implied mature-phase rate |
|---|---|---|---|---|
| V100 (2017) | — | $1.84 median | $0.97 median | **−27%/yr** geometric ([AIMultiple](https://aimultiple.com/gpu-index)) |
| A100 (2020) | ~$3–4 early | — | $1.62 median; lows $0.57–0.78 | sub-$1 open-market ([AIMultiple](https://aimultiple.com/gpu-index), [Vast.ai](https://vast.ai/pricing/gpu/A100-PCIE), [Thunder Compute](https://www.thundercompute.com/blog/ai-gpu-rental-market-trends)) |
| H100 (2022) | $7–10 (2023–24) | $2.58 mkt | $2.29–3.12 median | −28%/yr mature phase, +rebound 2026 |
| H200 | — | — | $3.39 median (spiked $2.29→$4.40 at Vast, May–Jun 2026) | volatile upward ([AIMultiple](https://aimultiple.com/gpu-index), [Thunder Compute](https://www.thundercompute.com/blog/ai-gpu-rental-market-trends)) |
| B200 / B300 (2024–25) | — | — | $5.24 / $6.99 median, **rising** | newest gen trends *up* first ([AIMultiple](https://aimultiple.com/gpu-index)) |

Pattern across three generations: **launch premium → crash as supply ramps and the next generation ships → −25 to −30%/yr grind → flatten at an operating-cost floor**. Notably, the newest generation's price goes *up* during its scarcity window — the decline only starts after supply catches demand.

---

## 4. The Long-Run Anchor: Price-Performance Research

Independent of cloud market noise, the underlying hardware economics set the trend slope:

- **FLOP/s per dollar doubles every ~2.5 years** across 470 GPUs (2006–2021); **every ~2.1 years** for ML-specific GPUs. ([Epoch AI — Trends in GPU price-performance](https://epoch.ai/blog/trends-in-gpu-price-performance))
- Across 20+ AI accelerators released 2012–2025, **compute per dollar improved ~30–40% per year**. ([Epoch AI — data insight](https://epoch.ai/data-insights/price-performance-hardware), [Our World in Data](https://ourworldindata.org/grapher/gpu-price-performance))

A 2.1–2.5-year doubling of compute-per-dollar is mathematically equivalent to the **price of a fixed amount of compute falling ~24–28% per year** — which is exactly the mature-phase rental decline observed empirically (V100 −27%/yr, H100 marketplace −28%/yr). The cloud market, once competitive, converges on the silicon trend.

---

## 5. So Is the Price Line Linear?

**No — and the distinction matters financially.** Three reasons the data rejects a straight line:

1. **The decline is proportional, not absolute.** H100 lost ~$6/hr in its first 18 months of correction, then ~$0.60/hr in the next 12. V100 lost $0.87/hr over two *years* at the same percentage rate. Equal *percentage* steps, shrinking *dollar* steps — that is exponential (geometric) decay, `P(t) = P₀·(1−r)^t`, not `P(t) = P₀ − kt`.
2. **There is a hard floor.** A rental price cannot fall below the host's marginal operating cost (electricity + bandwidth + maintenance + minimal margin). The market is already touching it: $0.15–0.18/hr for 4090-class hosts, ~$1.38–1.80/hr for H100 at efficient neoclouds. A linear line crosses zero; real prices flatten. Better functional form: **decay to a floor**, `P(t) = F + (P₀ − F)·e^(−kt)`.
3. **The path is volatile and can reverse.** Dec 2025–Jan 2026: +10% H100 spike. 2026: NVIDIA +20% rental hike, Vast.ai H100 +34% ($1.93→$2.58) in weeks, B300 the only line "still trending upward," and HBM supply constraints (≈40% of global DRAM output redirected to AI) re-tightening availability. ([Thunder Compute](https://www.thundercompute.com/blog/ai-gpu-rental-market-trends), [Crypto Briefing](https://cryptobriefing.com/nvidia-h100-rental-price-increase-2026/), [Kavout](https://www.kavout.com/market-lens/why-are-nvidia-h100-gpu-rental-prices-surging-by-40))

**Where the linear intuition is *approximately* right:** over a short window (2–4 years) in the mature phase, a geometric −25%/yr curve looks nearly straight, so a linear approximation doesn't badly mislead early on. It fails at the tails — it overstates how high prices stay in year 1–2 of a generation's correction (the crash is faster than linear) and overstates how low they go in years 5+ (the floor stops the slide).

### Recommended modeling form

For a revenue model on a fixed hardware cohort:

```
rate(t) = max( floor, rate₀ × (1 − d)^t )
  d = 20%/yr  base case   (consumer/near-floor cards, e.g. 4090-class)
  d = 30%/yr  stress case (matches flagship mature-phase observed decline)
  floor ≈ 1.5–2× host marginal opex per hour
```

### Implication for Superheat's assumptions

Superheat's existing model uses **−20%/yr base / −30%/yr stress** on compute pricing. Against this research:

- **−30%/yr stress is well-calibrated** — it matches the *worst observed sustained* decline for mature cards (V100 −27%/yr, H100 marketplace −28%/yr) and the fast end of the Epoch price-performance trend.
- **−20%/yr base is defensible and arguably conservative for consumer-class silicon**, which is the fleet-relevant tier: 4090 medians have been *stable to mildly declining* ($0.63 → $0.54 average on-demand) over the tracked window because these cards already sit near the floor, and the 4090→5090 generational step implies ~−20%/yr at constant tier.
- The model should additionally assume a **price floor** (host marginal cost ≈ $0.15–0.25/hr per 4090-class GPU at residential power rates) rather than letting the geometric decline run indefinitely — this is a *favorable* refinement, since Superheat's heat-offset economics effectively subsidize the electricity input that sets everyone else's floor.
- Volatility cuts both ways: the 2026 rebound (+10–34% moves) shows contracted/forward-sold capacity is worth modeling separately from spot.

---

## 6. References

1. Silicon Data — [H100 Rental Price Over Time (2023–2025): A Complete Market Analysis](https://www.silicondata.com/blog/h100-rental-price-over-time)
2. Silicon Data — [H100 Price Spike: Understanding the 10% Surge in GPU Rental Costs](https://www.silicondata.com/blog/h100-price-spike) (June 12, 2026)
3. IntuitionLabs — [H100 Rental Prices Compared: $1.49–$6.98/hr Across 15+ Cloud Providers](https://intuitionlabs.ai/articles/h100-rental-prices-cloud-comparison) (updated June 11, 2026)
4. CloudZero — [H100 GPU Cost in 2026: Buy, Rent, and Cloud Pricing Compared](https://www.cloudzero.com/blog/h100-gpu-cost/) (May 20, 2026)
5. AIMultiple — [Cloud GPU Rental Price Index](https://aimultiple.com/gpu-index) (58 providers, 17 GPUs, monthly snapshots Jul 2024–May 2026; updated May 20, 2026)
6. GetDeploying — [RTX 4090 Cloud Pricing: 14+ Providers](https://getdeploying.com/gpus/nvidia-rtx-4090) (updated June 12, 2026) and [H100 Cloud Pricing: 45+ Providers](https://getdeploying.com/gpus/nvidia-h100)
7. Thunder Compute — [AI GPU Rental Market Trends (June 2026)](https://www.thundercompute.com/blog/ai-gpu-rental-market-trends) (June 8, 2026)
8. Crypto Briefing — [Nvidia raises H100 rental prices by 20% in 2026](https://cryptobriefing.com/nvidia-h100-rental-price-increase-2026/) (May 20, 2026)
9. Kavout — [Why are NVIDIA H100 GPU rental prices surging by 40%](https://www.kavout.com/market-lens/why-are-nvidia-h100-gpu-rental-prices-surging-by-40)
10. SynpixCloud — [RunPod RTX 4090 Price Per Hour 2026](https://www.synpixcloud.com/blog/rtx-4090-cloud-rental-worth-it) (Feb 10, 2026; updated May 24, 2026)
11. Vast.ai — [RTX 4090 pricing](https://vast.ai/pricing/gpu/RTX-4090) · [A100 PCIe pricing](https://vast.ai/pricing/gpu/A100-PCIE)
12. RunPod — [RTX 4090 GPU Cloud](https://www.runpod.io/gpu-models/rtx-4090)
13. Epoch AI — [Trends in GPU Price-Performance](https://epoch.ai/blog/trends-in-gpu-price-performance) · [Performance per dollar improves ~30% each year](https://epoch.ai/data-insights/price-performance-hardware)
14. Our World in Data — [GPU computational performance per dollar](https://ourworldindata.org/grapher/gpu-price-performance)
15. JarvisLabs — [NVIDIA H100 Price Guide](https://jarvislabs.ai/blog/h100-price) · [A100 Price Guide](https://jarvislabs.ai/ai-faqs/nvidia-a100-gpu-price)

**Data caveats:** Aggregator medians mix on-demand/spot/reserved and SXM/PCIe form factors; hyperscaler list prices overstate what negotiated/committed buyers pay; marketplace lows ($0.08–0.17 4090, $0.47 H100) are often interruptible spot on unverified hosts. The 2026 NVIDIA rental-price-increase reports (Crypto Briefing, Kavout) are secondary press coverage — directionally corroborated by the Vast.ai/Verda escalations in Thunder Compute's data, but the exact "+20%/+40%" figures should be treated as reported, not audited.
