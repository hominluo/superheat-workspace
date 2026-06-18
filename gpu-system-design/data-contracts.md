# Superheat — Canonical Data Contracts & Glossary

**Status:** Draft v0.1 · **Date:** June 2026 · **Audience:** All engineering · **Authority:** NORMATIVE
**Scope:** This is the **single source of truth** for the shared objects that cross document and service boundaries: the reliability-tier enum, the reliability-score formula, the normalized job model, the telemetry frame, the signed usage record, the thermal constants, and the shared feasibility predicates. Where any other design doc previously defined these locally and divergently, **this doc wins** and the others must reference it.

> **Why this exists.** A cross-document review found the same first-class objects defined three different ways (reliability tier as `PLATINUM/GOLD/SILVER/BRONZE` vs `T1–T4` vs `spot/standard/premium`; the "normalized job" in two incompatible shapes; the telemetry frame with `heat_demand` as an enum in one doc and a wattage in another). Those docs each claimed to share a "lingua franca" that did not actually type-check. This doc defines that lingua franca once. **Other docs should link here, not redefine.**

---

## 1. Conventions

- **Units (SI-ish, explicit):** durations in **seconds** (`_s`) unless a field name says `_min`/`_h`; energy in **kWh**; power in **watts** (`_w`); memory in **MB** (`_mb`); money in **USD** as decimal (`_usd`); temperatures in **°C**. Never mix.
- **Identifiers:** `snake_case` field names; `lower_snake` enum values; UUIDs for internal entity ids; opaque strings for external refs.
- **Wire format:** structured messages are **protobuf v3** on the wire (`superheat.<thing>.v<major>`); JSON shown here is the human-readable projection. Schema version is carried in every message.
- **GPU model string — canonical form:** `"RTX4090"` (no space, no underscore). NOT `RTX_4090`, NOT `RTX 4090`. Enum: `RTX4090 | RTX5090 | OTHER`. (Telemetry/registry/job all use this exact token.)
- **Time:** UTC `timestamptz`; node clocks are NTP-synced and skew is reported in telemetry.

---

## 2. Reliability tier (canonical)

Reliability is a **single ordered 4-level ladder**, derived from the reliability **score** (§3). It is orthogonal to confidentiality (§2.2) — do not conflate `trusted_host` with a reliability tier.

| Tier (canonical) | Order | Score band | Meaning | Eligible work |
|---|---|---|---|---|
| `premium` | 3 | ≥ 85 | Seasoned, high-uptime, well-connected. | Contracted / non-interruptible / reserved + everything below |
| `standard` | 2 | 70–84 | Reliable enough for most marketplace work. | Standard marketplace + interruptible + filler |
| `spot` | 1 | 50–69 | Interruptible-only; flaky link or thermal availability. | Spot/interruptible + filler |
| `probation` | 0 | < 50 **or** unseasoned (< 30 d history) | New or failing; not tape-eligible. | Filler/backlog only; not financeable collateral |

**Comparison rule.** A job carries `reliability_tier_required ∈ {probation,spot,standard,premium}`. A node is eligible iff `tier_order(node) >= tier_order(job.reliability_tier_required)`. This is the ONE comparison used by the scheduler (`platform.md` §5), yield router (`platform.md` §C), and utilization engine (`platform.md` §5).

### 2.1 Crosswalk (retired local names)

Earlier (pre-consolidation) docs used three divergent vocabularies for this one concept; they are all retired in favor of the canonical tier. The schedulers, yield router, and utilization logic that used them now live together in `platform.md` and use the canonical names.

| Canonical | Retired aliases |
|---|---|
| `premium` | PLATINUM · T1 Reliable |
| `standard` | GOLD · T2 Standard |
| `spot` | SILVER · T3 Interruptible-only |
| `probation` | BRONZE · T4 Probation |

### 2.2 Confidentiality class (orthogonal axis)

`confidentiality_class ∈ {standard, trusted_host}`. This is a **security/hardware** property (see `security-and-compliance.md`), NOT reliability. `trusted_host` = vetted commercial site / TEE-capable hardware. A node has both a `reliability_tier` and a `confidentiality_class`; a job may require either or both. `trusted_host` must never appear in `reliability_tier_required`.

---

## 3. Reliability score (canonical formula)

Score ∈ **[0, 100]**, computed per node over a trailing 30-day window. **Positive weights sum to 1.0 so a perfect node reaches 100** (the previous formula summed to 0.80, making the top tier mathematically unreachable — fixed here).

```
uptime_frac      ∈ [0,1]   # fraction of expected heartbeats received, EXCLUDING control-plane-down windows (see note)
thermal_avail    ∈ [0,1]   # fraction of time the node could run compute when asked (thermal window open)
net_quality      ∈ [0,1]   # min(1, measured_useful_uplink_mbps / TARGET_UPLINK_MBPS), blended w/ latency/loss
preempt_penalty  ∈ [0,1]   # norm(involuntary_preemptions_per_day / PREEMPT_CAP), clamped
fault_penalty    ∈ [0,1]   # norm(xid_errors + thermal_trips + ckpt_failures per day / FAULT_CAP), clamped

base  = 0.45*uptime_frac + 0.25*thermal_avail + 0.20*net_quality + 0.10*(1 - preempt_penalty)   # ∈ [0,1], =1 when perfect
score = round( 100 * base * (1 - 0.30*fault_penalty) )                                            # faults scale down, max -30%
```

- Perfect node (`uptime=thermal=net=1`, `preempt=fault=0`) → `score = 100` → `premium`. ✓
- `norm(x) = min(1, x)`. `TARGET_UPLINK_MBPS`, `PREEMPT_CAP`, `FAULT_CAP` are tunables published in the registry config.
- **Seasoning gate:** regardless of score, a node with < 30 days of clean history is `probation` and **not tape-eligible** (`platform.md` §7 tape view filters `seasoned=true`).
- **Control-plane-down exclusion (fixes the false-outage bug):** heartbeat gaps caused by a *cloud/aggregator* outage (correlated across many nodes) are excluded from `uptime_frac`; only node-attributable gaps count. The aggregator stamps `cloud_degraded` windows.

---

## 4. Normalized job model (canonical)

This is the lingua franca between connectors, the yield router, the scheduler, and the ledger. Connectors (`platform.md` §A) translate each marketplace's native job into this; the scheduler (`platform.md` §5) consumes exactly this. **One schema, nested, seconds for time.**

```jsonc
{
  "schema": "superheat.job.v1",
  "job_id": "uuid",                      // internal id
  "source": "vast",                      // enum: vast|runpod|salad|ionet|akash|direct|reserved|backlog
  "source_job_ref": "vast:12345",        // opaque external id (null for own demand)
  "tenant_class": "marketplace",         // marketplace|direct|reserved|internal_backlog
  "compute": {
    "gpu_model": "RTX4090",              // canonical token (§1)
    "gpu_count": 1,                       // 1..8 ; >1 requires a multi-GPU-capable node
    "min_vram_mb": 24000,
    "expected_duration_s": 7200,          // SECONDS (estimate; null = open-ended, e.g. inference serving)
    "image_ref": "cr.superheat/...@sha256:..."
  },
  "scheduling": {
    "interruptible": true,                // false = non-interruptible; gates the window-fit predicate (§7)
    "reliability_tier_required": "spot",  // §2 enum
    "min_thermal_window_s": 1800,         // SECONDS of contiguous runnable time required to start
    "max_start_latency_s": 300
  },
  "economics": {
    "gross_price_usd_per_gpu_hr": 0.40,   // what the buyer pays
    "platform_fee_frac": 0.25,            // marketplace take (0 for direct/backlog)
    "net_price_usd_per_gpu_hr": 0.30      // gross*(1-fee) ; the number the yield router compares
  },
  "trust": {
    "confidentiality_required": "standard", // standard|trusted_host (§2.2)
    "needs_tee": false,
    "egress_policy": "restricted",
    "no_persistence": true
  }
}
```

Field-name notes that retire local variants: use `reliability_tier_required` (not `min_tier`), `expected_duration_s` in seconds (not `expected_duration_s` vs `expected_runtime_min`), `net_price_usd_per_gpu_hr` (not `bid_usd_per_gpu_hr`), `gpu_model:"RTX4090"`. The `source` enum is the full set `{vast,runpod,salad,ionet,akash,direct,reserved,backlog}` — all docs use this exact set.

---

## 5. Telemetry frame (canonical)

One frame, emitted by the node agent (`node.md` §5), consumed by the control plane (`platform.md` §2). `heat_demand` is BOTH a mode enum AND a wattage — the prior enum-vs-watts conflict is resolved by carrying both.

```jsonc
{
  "schema": "superheat.telemetry.v1",
  "node_id": "uuid",
  "agent_version": "1.0.0",
  "seq": 84211,                         // monotonic; gaps = lost samples
  "ts": "2026-06-18T12:00:00Z",
  "clock_skew_ms": 12,
  "gpus": [                             // ALWAYS an array, even for H1 (length 1)
    { "slot": 0, "model": "RTX4090", "util_pct": 96, "mem_used_mb": 21500,
      "power_w": 431, "power_cap_w": 450, "temp_c": 71, "xid_errors": 0 }
  ],
  "thermal": {
    "mode": "storing",                  // none|realtime|storing|full  (enum)
    "heat_demand_w": 1200,              // instantaneous useful heat demand, WATTS (numeric)
    "tank_temp_c": [54.0, 49.0],        // stratification zones
    "storage_headroom_kwh": 4.1,        // remaining absorbable heat (see §6)
    "comfort_ok": true
  },
  "board_power_w": 610,                 // whole-node wall draw (for circuit-budget enforcement)
  "net": { "path": "direct", "uplink_mbps": 28.0, "downlink_mbps": 180.0, "rtt_ms": 22, "loss_pct": 0.1 },
  "ckpt_age_s": 95,
  "cloud_degraded": false,              // set by aggregator on correlated outage (excluded from uptime)
  "sig": "ed25519:..."                  // node identity key (§8 of node.md)
}
```

Canonical decisions: memory in **MB** (`mem_used_mb`), GPUs always an **array** keyed `gpus` (not `gpu`), net path under `net.path ∈ {direct,relay}` (not both `net.overlay` and `net.relay`), heat demand carried as `thermal.mode` + `thermal.heat_demand_w`.

---

## 6. Thermal constants (canonical)

The tank-as-battery math (`platform.md` §2, `platform.md` §6) must use ONE capacity number, derived and SKU-anchored — not a hardcoded `12.0`.

```
Specific heat of water  c = 4.186 kJ/(kg·°C)
80 US gal ≈ 302.8 kg
Thermal capacitance     C_th = 302.8 * 4.186 / 3600 ≈ 0.352 kWh/°C
Usable temperature band ΔT_usable ≈ 25 °C  (e.g. 40→65 °C before comfort/scald limits)
Usable thermal storage  E_store = C_th * ΔT_usable ≈ 8.8 kWh   ← CANONICAL for the standard 80-gal H1 tank
Heat-capture efficiency η_capture ≈ 0.95  (PLACEHOLDER — must be measured in Phase A; flagged in node.md)
```

- **Canonical `E_store` ≈ 8.8 kWh** for the standard 80-gal tank. The control-plane registry must store `usable_thermal_kwh` **per SKU** (the prior hardcoded `12.0` is retired) and the thermal coordinator imports it. Commercial/larger-tank SKUs carry their own value.
- `storage_headroom_kwh` in telemetry (§5) = `E_store * (T_max - T_current_avg)/ΔT_usable`, clamped ≥ 0.
- `η_capture = 0.95` is an unvalidated placeholder; every duty-cycle conclusion that uses it is conditional until measured.

---

## 7. Shared feasibility predicate (canonical)

The "don't put a non-interruptible job on a closing thermal window" rule is defined ONCE here and referenced by scheduler, yield router, and utilization engine (which previously each had a slightly different check).

```
fits(job, node, window_remaining_s):
    if tier_order(node.reliability_tier) < tier_order(job.scheduling.reliability_tier_required): return false
    if job.trust.confidentiality_required == "trusted_host" and node.confidentiality_class != "trusted_host": return false
    if job.compute.gpu_count > node.free_gpus: return false
    if not job.scheduling.interruptible:
        # non-interruptible: the whole job must fit the currently-open thermal window
        if job.compute.expected_duration_s is None: return false        # open-ended can't be non-interruptible
        if window_remaining_s < job.compute.expected_duration_s: return false
    else:
        # interruptible: only require enough window to be worth starting + checkpoint-safe
        if window_remaining_s < max(job.scheduling.min_thermal_window_s, T_GRACE_S): return false
    return true
```

- **`T_GRACE_S = 30`** (canonical). The graceful-stop budget for an interruptible job to flush its in-flight checkpoint. `connectivity-and-data.md` owns the checkpoint contract; `platform.md`'s prior "30–60 s" is retired in favor of 30 s.
- Because a 30 s grace cannot finish a multi-minute upload, the **durable** resume path is always the last PoP-persisted checkpoint, not the in-flight flush (see `connectivity-and-data.md`).

---

## 8. Signed usage record (canonical metering primitive)

The billing/securitization tape (`platform.md` §7) is built from **double-signed usage records**. This defines the producer side (the node emits it; `node.md` must implement it) and the cloud counter-signature, so the tape has a real provenance chain.

```jsonc
{
  "schema": "superheat.usage.v1",
  "usage_id": "uuid",
  "node_id": "uuid",
  "gpu_slot": 0,                        // which physical GPU earned (per-GPU, not per-node)
  "job_id": "uuid",
  "source": "vast",
  "start_ts": "...", "end_ts": "...",   // a usage record NEVER spans a billing-period boundary; split at boundary
  "gpu_seconds": 3600,
  "energy_kwh": 0.43,                   // metered wall energy attributed to this GPU (for net-margin/opex)
  "gross_usd": 0.40, "fee_usd": 0.10, "net_usd": 0.30,
  "node_sig": "ed25519:...",            // node identity key signs the metered facts
  "cloud_sig": "ed25519:..."            // control plane counter-signs after reconciliation
}
```

- **Per-GPU, not per-node** — `gpu_slot` is mandatory so the tape view aggregates by GPU and does not Cartesian-multiply revenue by GPU count (the prior tape SQL join bug). The canonical aggregation joins `usage_record` to `gpu` on **both** `node_id` AND `gpu_slot`.
- **Period-boundary split:** a record never crosses a month boundary; long jobs emit one record per period. This keeps monthly `utilization_pct` correct.
- **Reconciliation:** `net_usd` summed over a marketplace must reconcile to that marketplace's settlement statement (`platform.md` Part D). The ±tolerance applies to the **signed usage records vs marketplace settlement**, NOT to lossy telemetry — telemetry is for health/scoring, usage records are for money. (This corrects the prior "reconcile lossy telemetry within ±2%" claim.)

---

## 9. How to use this doc

- Other docs **reference** these definitions ("see `data-contracts.md` §4 for the job schema") rather than restating them. Where a doc needs to show a projection (e.g. a flat DB row), it must note it is a projection of the canonical object and map the fields.
- Changes here are breaking by definition: bump the `vN` schema tag and update the crosswalk.
- Open tunables to pin in the registry config: `TARGET_UPLINK_MBPS`, `PREEMPT_CAP`, `FAULT_CAP`, per-SKU `usable_thermal_kwh`, `T_GRACE_S` (=30), reconciliation tolerance.

*Cross-references:* `platform.md` (registry, reliability scoring, scheduler, tape), `platform.md` (connectors, normalized job, yield router, settlement), `platform.md` (thermal constants, feasibility), `node.md` (telemetry + usage producer, identity key), `node.md` (per-SKU thermal/electrical constants), `security-and-compliance.md` (confidentiality class, signing keys).
