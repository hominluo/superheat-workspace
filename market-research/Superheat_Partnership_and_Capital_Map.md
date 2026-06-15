# Superheat — Partnership & Capital Map: Loan Partners, Liquidity Providers, and the Right Investors for This Stage

*v0.3 — internal working draft, June 2026. Doc 3 of 3.*
*Builds on: `Superheat_Compute-Network_Debt-Engine_Onepager.md` (the thesis) and `Superheat_Recalculated_Economics_and_Debt_Business_Plan.md` (the numbers + funding ladder). All named-deal facts are sourced (links at end); fit assessments are our judgment.*

---

## 0. Why not VCs, and who instead

The business we're building is an **asset originator**: value accrues from (1) cheap, reliable leverage against the fleet, (2) origination volume, and (3) a retained servicing/residual annuity. Generalist VCs price software growth, not assets — they can't provide the leverage, they push for burn-funded growth that this model punishes (every unit funded with equity instead of debt destroys the ROE story), and the monoline solar originators that leaned on equity + assumed ABS take-out are the cautionary tales (§8). What the cap table needs now:

| Capital need | Right counterparty type | Section |
|---|---|---|
| Residential loan origination (no balance sheet) | Point-of-sale home-improvement lenders | §1 |
| Warehouse leverage for the commercial TPO fleet | Securitized-products platforms, banks, ABF funds | §2 |
| Take-out / scale liquidity | Forward-flow whole-loan buyers, insurance balance sheets | §3 |
| GPU-literate debt | The neocloud lender club (now a $20B+ market) | §4 |
| Equity / first-loss / FOAK capital | Hybrid credit+equity funds, sustainable-infra platforms, strategics | §5 |
| Contracted demand + hedges (rating fuel) | Compute offtakers, futures markets, VPP aggregators | §6 |

---

## 1. Residential loan origination partners (the "GoodLeap seat")

Goal: a partner whose paper already securitizes, whose contractor network already installs water heaters, and who takes zero Superheat balance-sheet risk. Dealer-fee economics flow to Superheat as manufacturer-contractor; the loan is theirs.

| Partner | Why them (evidence) | Status / caution | The ask |
|---|---|---|---|
| **GoodLeap** ★ | Largest US residential sustainable-home lender — **$30B+ originated, 1M+ homeowners**; finances HVAC/water-efficiency today; active ABS shelf (2026-1 just priced, §Doc 2) | Alive and issuing; named in MN AG dealer-fee suit (2024) — keep fee structures transparent | Add Superheat H1 as an approved product class; pilot 500 loans through their contractor app |
| **GreenSky** (Sixth Street-led consortium) | Pure home-improvement POS platform; bought from Goldman **March 2024 by Sixth Street + institutional investors**; KBRA-rated shelf still printing (2025-3) | Healthy; consortium owner = potential warehouse/FF adjacency (Sixth Street, §3) | Same — merchant/program agreement for H1 installs |
| **Service Finance Co. (Truist)** | The HVAC dealer-financing incumbent — exactly our installer channel | Bank-owned, conservative | Dealer program for Superheat-certified plumbers/HVAC contractors |
| **Synchrony / Wells Fargo Retail Services** | Big-box and OEM-branded installment programs (they run financing for major HVAC OEMs) | Slower, high volume requirements | OEM-style private-label program once volume justifies |
| **Wisetack / Hearth / Acorn Finance** | Fintech POS for home-services contractors; fastest integration, API-first | Smaller ticket appetite | Immediate-term: financing button in the Superheat quote flow while courting GoodLeap |
| **Foundation Finance, Medallion Bank** | Home-improvement dealer paper specialists (incl. sub-prime tiers) | Niche | Secondary capacity / credit-tier coverage |
| **Cross River, WebBank, Celtic Bank** | Partner banks that rent charters to fintech originators | For Phase-2+ captive origination | Bank-partnership term sheet when we take origination in-house |

**Dead pool — structure lessons, not partners:** Solar Mosaic (**Ch. 11, June 2025** — rate spike + failed refinancing), Sunlight Financial (**Ch. 11, Oct 2023**), Sunnova (Ch. 11, June 2025). Common failure: monoline reliance on continuous ABS take-out, opaque dealer fees (the **Minnesota AG suit** against GoodLeap/Sunlight/Mosaic/Dividend over $35M of hidden fees), and no second revenue leg. Superheat's counter: the financed asset *produces income*, fees disclosed, and origination is a complement to hardware margin — not the whole company.

---

## 2. Warehouse & liquidity providers (the commercial fleet's first lever)

| Partner | Why them (evidence) | The ask |
|---|---|---|
| **ATLAS SP Partners (Apollo)** ★ | *The* esoteric-warehouse platform: ex-Credit Suisse securitized-products group, Apollo-majority with **ADIA cornerstone** and a **$5B BNP Paribas collaboration**; printing first-time-originator facilities *right now* (Wayflyer $250M and Fundbox/Blue Owl extensions, **Feb 2026**) | $25–50M revolving warehouse against seasoned commercial units (70–75% advance, SOFR+300–450), upsizable |
| **Jefferies, Barclays, Goldman, Citi, CIBC** | Goldman/CIBC/Citi are GoodLeap's bookrunners — they already underwrite the closest analog asset; Jefferies/Barclays lead esoteric first-timers | Warehouse + future ABS left-lead mandate (dangle the inaugural deal) |
| **MUFG, Natixis, Société Générale** | Project-finance banks active in distributed energy and (SocGen) publishing on heat-recovery economics | Second warehouse / EU expansion facility |
| **Texas Capital, Western Alliance, First Citizens (ex-SVB)** | Regional specialty-finance lender group for sub-$50M first facilities | Cheaper small first facility if ATLAS terms are heavy |
| **Victory Park, Crayhill, Fortress, Ares Alternative Credit, KKR ABF, Värde, Neuberger Specialty Finance** | Private-credit ABF desks that do hybrid warehouse + mezz for new asset classes | Unitranche warehouse incl. the first-loss strip if banks balk at FOAK collateral |

---

## 3. Forward-flow / whole-loan buyers (residential paper at scale)

| Partner | Why them (evidence) | The ask |
|---|---|---|
| **Castlelake** ★ | The most active forward-flow buyer in consumer credit: **Upstart up to $1.2B (Jun 2024) → $4B (Oct 2024) → third $1.5B agreement (Nov 2025)**; **Pagaya $1B (2024) + $2.5B (2025)**; **Alma €3B+** | 12-month forward flow for H1 home-improvement loans at par + servicing retained, renewable |
| **Blue Owl (Atalaya)** | Bought Atalaya (esoteric ABF pioneer); already co-lending next to ATLAS (Fundbox, Feb 2026) | Same, as second committed buyer (never single-source the take-out) |
| **Sixth Street** | Owns GreenSky — natural buyer of paper originated through its own platform | Bundled origination + flow deal via GreenSky |
| **Fortress, Ares, PIMCO, Carlyle Credit, Centerbridge** | Standing whole-loan programs across consumer/specialty | Backup flow capacity; price discovery |
| **Insurance balance sheets** — Athene (Apollo), Global Atlantic (KKR), **Eldridge** (in CoreWeave's lender group already), MassMutual/Barings, MetLife IM, Guggenheim | The end-state senior buyers; Eldridge has *already* underwritten GPU collateral | Anchor orders for the inaugural ABS senior class (Phase 3) |

---

## 4. GPU-literate lenders & strategic compute capital

The neocloud debt market — **$20B+ outstanding, GPU-collateralized** — has produced a lender club that has already priced this collateral:

| Partner | Evidence | Relevance to Superheat |
|---|---|---|
| **Magnetar + Blackstone** | Led CoreWeave's $2.3B (2023) and **$7.5B (2024)**; the franchise GPU-credit duo | The call to make when the commercial fleet passes ~$50M; they understand utilization-based underwriting |
| **Macquarie** | **$500M GPU-backed facility to Lambda** | Equipment-finance DNA + GPU comfort — strong warehouse alternative |
| **CoreWeave lender syndicate**: Coatue, Carlyle, CDPQ, DigitalBridge Credit, BlackRock funds, Eldridge, Great Elm | Participated in the $7.5B facility; Moody's rated the successor **$8.5B DDTL A3 (Mar 2026)** against clusters + **$19.2B contract backlog** | The full distribution list for Superheat's eventual rated paper |
| **NVIDIA** | NVentures deployed **~$1B/yr** into AI startups; holds ~11% of CoreWeave and **backstops $6.3B of unsold capacity through 2032**; leased 18K GPUs from Lambda for $1.5B | Three asks: Inception membership (now), NVentures strategic equity (Phase 2), and — the big one — a **capacity backstop on fleet GPU-hours**, the single most rating-positive contract Superheat could sign |
| **Dell Financial Services / HPE Financial Services** | As-a-service GPU leasing programs; Dell supplying GB300 racks to neoclouds | Lease (don't buy) the fleet's compute modules → shifts CAPEX off the SPV, simplifies the year-5 refresh covenant |
| Residual-value datum | **Used H100s traded at 60–70% of original price in 2025** | Use as the collateral-recovery anchor in lender models (consumer cards recover less — underwrite at 30–40%) |

---

## 5. The right equity / first-loss investors at this stage (the VC replacement)

**5a. Hybrid credit-plus-equity funds — purpose-built for asset-originating startups:**

| Investor | Evidence of fit | The ask |
|---|---|---|
| **Upper90** ★ | Hybrid debt+equity, **$2.2B+ deployed**; was an **initial institutional capital partner to Crusoe Energy** — the canonical energy-compute crossover — plus Octane (powersports lending) | Lead a $10–20M equity + first-loss facility for the Phase-1 fleet |
| **CoVenture** | Explicit "emerging assets" asset-based credit mandate | Co-anchor; they underwrite assets banks won't touch yet |
| **i80 Group** | ABF for fintech/proptech originators between venture and bank stages | Warehouse-adjacent first-loss capital |
| **Architect Capital, Tacora** | Early asset-based capital for novel originators | Smaller complementary tickets |

**5b. Sustainable-infrastructure project capital — already funding compute-energy hybrids:**

| Investor | Evidence of fit | The ask |
|---|---|---|
| **Generate Capital** ★ | **Up to $100M credit facility to Soluna (Sept 2025)** — renewable-powered bitcoin/AI compute; owns-and-operates distributed energy; invented the energy-as-a-service playbook Superheat's HSA copies | Programmatic project equity: they fund unit deployments, Superheat operates (their standard model) |
| **Spring Lane Capital** | Hybrid project + growth capital for the **"missing middle"**; funded **Soluna's Project Dorothy** bitcoin-hosting build | $20–30M project-equity commitment Phase 1–2 (their sweet spot, per their 7 Generation EV deal) |
| **HASI** | Public sustainable-infra investor in distributed energy + efficiency; DOE Better Buildings partner | Phase-2/3 programmatic capital + credibility for the rating process |
| Greenbacker, Excelsior Energy Capital | Distributed-energy fund peers | Fill out the project-equity syndicate |

**5c. Strategic corporates — the ones that have already paid for this exact thesis:**

| Strategic | Evidence | Angle |
|---|---|---|
| **Octopus Energy** ★ | **£200M into Deep Green (Jan 2024)** — UK data-center-heat-reuse (pools, district heat); runs Kraken platform across millions of homes | The single most obvious strategic investor/JV for EU/UK residential; could white-label H1 to its customer base |
| **Centrica / British Gas** | **Live trial with Heata (Feb 2025)** — compute units on domestic hot-water cylinders, free hot water for households | UK utility pilot partner; Heata validates but is ARM-server scale — Superheat's GPU economics are an order bigger |
| **A2A (Milan) / ADEME (France)** | A2A integrates **Qarnot's** digital-boiler heat into district heating; ADEME Investissement funded Qarnot's €35M round | EU district-heating + public-capital templates |
| **Rheem** | Partners with **EnergyHub (Mercury DERMS, 30+ utilities)** and **Renew Home** on grid-interactive water heaters | Manufacturing + distribution + DR enrollment in one partner; alternatively A. O. Smith, Bradford White, **Rinnai/Navien (Korea manufacturing base)**, Ariston, NIBE, Bosch |
| **KEPCO / Tokyo Gas / Osaka Gas innovation arms** | East-Asia utility venture programs; Superheat already exhibits alongside KEPCO at trade shows | Korea/Japan market entry + manufacturing adjacency |
| Engie New Ventures, National Grid Partners, Constellation Tech Ventures, EDF Pulse | US/EU utility CVCs with distributed-energy mandates | Round participants + program channels |

---

## 6. Demand-side & risk-transfer partners (manufacture *contracted* revenue)

Every contracted GPU-hour and hedged dollar moves the ABS rating up and the coupon down:

- **Marketplaces (multi-home the fleet):** Vast.ai (current; its **Vast Finance** program proves lenders already finance hosts on its rails), RunPod, Salad, io.net, Akash, Prime Intellect, Foundry, Together — covenant target: no platform >60% of fleet revenue.
- **Compute offtake anchors:** enterprise inference aggregators (OpenRouter-style), regional AI labs, and — the prize — an **NVIDIA-style capacity backstop** (the CoreWeave/$6.3B precedent) or a Google/Fluidstack-style contracted-revenue wrap (~$6.7B template).
- **Compute futures / forwards:** **SF Compute**'s market for forward-selling GPU-hours — sell 12–24-month strips on a fleet slice = instant contracted revenue line for rating agencies.
- **Bitcoin-mode floor hedging:** **Luxor hashprice forwards** — hedge the ASIC/fallback revenue floor on units that retain mining capability (the original H1 fleet).
- **VPP / demand-response aggregators:** **EnergyHub** (already aggregates Rheem water heaters for 30+ utilities — the integration is literally pre-built for our device class), **Renew Home** (Google-backed, Rheem CA/NY programs), Voltus, CPower, Leap — $20–100/unit-yr of utility-grade contracted income (Doc 2 §3.7).

---

## 7. The first ten calls (priority order, with the specific ask)

1. **Upper90** — lead Phase-1 hybrid equity + first-loss ($10–20M). *They did Crusoe; thirty-second pitch.*
2. **Generate Capital** — programmatic project capital for commercial TPO units. *They just did Soluna's facility.*
3. **GoodLeap** — H1 as an approved financed product; 500-loan pilot. *Zero balance sheet, instant residential motion.*
4. **ATLAS SP** — $25–50M warehouse term sheet for the seasoned commercial fleet. *Active with first-time originators this quarter.*
5. **Castlelake** — forward-flow LOI for residential paper contingent on 12-month tape. *Three Upstart renewals say they renew when tape performs.*
6. **NVIDIA (Inception → NVentures)** — ecosystem membership now; backstop conversation when fleet >10K GPUs.
7. **Octopus Energy** — UK/EU strategic round + Kraken-channel pilot. *They wrote £200M for a weaker version of this thesis.*
8. **Spring Lane** — $20–30M project equity to bridge the "missing middle."
9. **EnergyHub** — fleet-wide DR/VPP enrollment partnership (Rheem integration as template).
10. **Macquarie** — GPU-collateral warehouse alternative; keeps ATLAS honest on terms.

Parallel track (no call needed yet, keep warm): Sixth Street/GreenSky, Blue Owl, Magnetar/Blackstone, Eldridge, SF Compute, Rheem/Navien.

---

## 8. Anti-pattern appendix — why the dead died (and our counters)

| Failure | Who | Counter in our structure |
|---|---|---|
| Monoline dependence on continuous ABS take-out | Mosaic (Ch.11 6/2025), Sunlight (Ch.11 10/2023), Sunnova (Ch.11 6/2025) | Hardware margin + servicing + compute revenue are independent legs; growth gated by *committed* facilities (Doc 2 §3.8) |
| Hidden dealer fees → AG suits, reputational ABS taint | MN AG vs GoodLeap/Sunlight/Mosaic/Dividend ($35M hidden fees) | Transparent published dealer fee; income-producing collateral reduces the need for fee-stuffing |
| Rate-spike duration mismatch | Mosaic's failed refinancing | 5–6-yr amortizing structures matched to GPU life, not 25-yr paper funded short |
| Collateral that only *costs* the homeowner | All solar-loan books | The H1 *pays* the borrower — payment offset is the underwriting feature no solar lender ever had |

---

## Sources

**Origination / POS lending**
- [GoodLeap — company overview ($30B+, 1M+ homeowners)](https://en.wikipedia.org/wiki/GoodLeap) · [Mosaic Ch.11 case summary (June 2025)](https://bondoro.com/mosaic/) · [Mosaic bankruptcy detail — $45M DIP, $8B portfolio](https://elevenflo.com/blog/mosaic-sustainable-finance-bankruptcy) · [GreenSky sold to Sixth Street-led consortium (Mar 2024)](https://www.credible.com/personal-loan/greensky-personal-loans-review) · [GreenSky home-improvement program](https://www.greensky.com/home-improvement/) · [KBRA — GreenSky 2025-3 ratings](https://finance.yahoo.com/news/kbra-assigns-preliminary-ratings-greensky-194700992.html) · [MN AG suit over dealer fees](https://pv-magazine-usa.com/2024/04/26/minnesota-sues-goodleap-sunlight-mosaic-and-dividend-over-dealer-fees/) · [TIME — solar lending fraud risks](https://time.com/6982203/solar-lending-fraud/)

**Warehouse / securitized products**
- [ATLAS SP Partners](https://www.atlas-sp.com/index.html) · [Apollo — ATLAS expansion with ADIA cornerstone](https://ir.apollo.com/news-events/press-releases/detail/451/apollo-announces-expansion-of-atlas-sp-partners-and-abf) · [ATLAS × BNP Paribas $5B collaboration](https://usa.bnpparibas/en/apollos-atlas-sp-and-bnp-paribas-announce-5-billion-strategic-collaboration/) · [Wayflyer $250M facility (Feb 2026)](https://fintech.global/2026/02/19/fintech-wayflyer-secures-250m-credit-facility/) · [Fundbox + ATLAS + Blue Owl (Feb 2026)](https://fintech.global/2026/02/20/fundbox-expands-credit-facility-with-atlas-and-blue-owl/)

**Forward flow**
- [Castlelake × Upstart $1.2B (Jun 2024)](https://www.prnewswire.com/news-releases/castlelake-agrees-to-purchase-up-to-1-2-billion-of-consumer-installment-loans-originated-on-upstarts-platform-302177888.html) · [Castlelake × Upstart $4B (Oct 2024)](https://www.castlelake.com/article/castlelake-reaches-purchase-agreement-for-up-to-4-billion-of-consumer-installment-loans-originated-on-upstarts-platform/) · [Upstart — third Castlelake forward flow, $1.5B (Nov 2025)](https://ir.upstart.com/news-releases/news-release-details/upstart-announces-15b-forward-flow-agreement-castlelake) · [Castlelake × Pagaya $1B](https://www.castlelake.com/article/pagaya-and-castlelake-announce-forward-flow-agreement-to-purchase-up-to-1-billion-of-consumer-loans-originated-on-the-pagaya-network/) · [Pagaya × Castlelake $2.5B (Jul 2025)](https://investor.pagaya.com/news-releases/news-release-details/pagaya-signs-new-forward-flow-agreement-castlelake-purchase-25) · [Castlelake × Alma €3B](https://www.prnewswire.com/news-releases/castlelake-announces-forward-flow-agreement-with-alma-to-purchase-over-3-billion-of-consumer-loans-302270563.html)

**GPU asset-class finance**
- [The AI Insider — Who is financing the new GPU asset class (Jun 10, 2026): Macquarie/Lambda $500M, Nscale $1.4B, Fluidstack/Google ~$6.7B contracts, NVIDIA $6.3B backstop, used-H100 residuals 60–70%](https://theaiinsider.tech/2026/06/10/guest-post-who-is-financing-the-new-gpu-asset-class/) · [Blackstone — CoreWeave $7.5B](https://www.blackstone.com/news/press/coreweave-secures-7-5-billion-debt-financing-facility-led-by-blackstone-and-magnetar/) · [CoreWeave — $8.5B IG-rated DDTL](https://investors.coreweave.com/news/news-details/2026/CoreWeave-Closes-Landmark-8-5-Billion-Financing-Facility-Achieving-First-Investment-Grade-Rated-GPU-backed-Financing/default.aspx) · [Lambda $480M equity at $4B (NVIDIA-backed)](https://techfundingnews.com/nvidia-backed-lambda-lands-480m-at-4b-valuation-to-scale-its-ai-cloud/) · [Neocloud GPU-backed debt >$20B](https://davefriedman.substack.com/p/neoclouds-hold-more-than-20-billion)

**Sustainable-infra & strategic capital**
- [Soluna × Generate Capital — up to $100M (Sept 2025)](https://www.solunacomputing.com/news/soluna-secures-30m-from-spring-lane-capital-to-fuel-project-dorothy-2/) · [Spring Lane Capital — hybrid project/growth "missing middle"](https://www.springlanecapital.com/) · [Spring Lane × 7 Generation $20M project equity](https://www.prnewswire.com/news-releases/spring-lane-capital-announces-20m-project-equity-commitment-to-7-generation-capital-to-finance-electric-vehicle-ev-projects-for-commercial-fleets-301315143.html) · [HASI — distributed energy & efficiency investor](https://www.hasi.com/) · [Upper90 — hybrid credit+equity, $2.2B+, Crusoe early partner](https://upper90.io/) · [CoVenture — asset-based credit for emerging assets](https://www.coventure.vc/asset-based-credit) · [i80 Group](https://www.i80group.com/)

**Heat-reuse compute precedents & utility strategics**
- [Centrica/British Gas × Heata trial (Feb 2025)](https://www.centrica.com/media-centre/news/2025/british-gas-partners-with-heata-on-trial-to-reuse-waste-heat-from-data-processing/) · [Octopus Energy — £200M into Deep Green (Jan 2024)](https://octopus.energy/press/deep-green-investment/) · [The Register — Deep Green £200M](https://www.theregister.com/2024/01/16/deep_green_gets_200m_from/) · [Qarnot €35M raise](https://tech.eu/2023/01/11/reclaiming-data-centre-waste-heat-nets-qarnot-eur35-million/) · [Qarnot × A2A district heating (Brescia)](https://www.datacenterdynamics.com/en/news/qarnot-launches-italian-data-center-waste-heat-to-be-used-in-local-district-heating-network/) · [ADEME Investissement — Qarnot](https://www.ademe-investissement.fr/en/our-transactions/energy/qarnot-computing/)

**VPP / demand response**
- [EnergyHub × Rheem grid-interactive water heating](https://www.energyhub.com/blog/rheem-integration/) · [United Illuminating × EnergyHub × Rheem program](https://www.rheem.com/about/news-releases/united-illuminating-announces-successful-income-eligible-water-heater-program-in-partnership-with-energyhub-and-rheem/) · [Rheem × Renew Home](https://www.rheem.com/water-heating/articles/introducing-rheems-partnership-with-renew-home/)

*Sunnova Ch. 11 (June 2025) from general market knowledge; all other distress events sourced above. Fit assessments and ask sizes are our judgment, not reported facts. Verify current fund mandates before outreach — ABF mandates shift quarterly.*
