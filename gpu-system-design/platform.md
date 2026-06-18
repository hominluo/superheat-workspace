# Superheat — Cloud Platform (Control Plane, Orchestration, Demand & Renter API)

**Status:** Draft v0.1 · **Date:** June 2026 · **Audience:** Engineering (+ direct/enterprise renters for Part D)
**Scope:** This document is the consolidated platform deep-dive. It specifies everything that turns scattered heater-embedded GPU boxes into one fleet, routes every node-hour to its highest-yield buyer, drives idle toward zero, and exposes a clean external surface to direct renters. It expands §6 ("Control plane / cloud"), §7 ("Demand layer — hybrid routing"), §8 ("Driving utilization toward ~zero idle"), and §11 ("own the demand & UX") of `system-design-overview.md`.

The platform is presented in four Parts that share one set of constraints, schemas, and cross-references:

- **Part A — Control plane:** registry & inventory, telemetry, OTA, reliability scoring, scheduler, thermal coordinator, metering ledger / securitization tape, scale architecture (§1–§10).
- **Part B — Demand & yield routing:** marketplace connectors, the normalized job, own demand, the yield router, the concentration guard, settlement reconciliation (§11–§19).
- **Part C — Utilization engine:** tank-as-battery model, thermal windows, the no-idle four-layer stack, reliability tiering, preemption, seasonality, fleet-blend reconciliation (§20–§30).
- **Part D — Renter API & SDK:** the external REST/gRPC surface, the SDK and CLI, the job lifecycle, tier/confidentiality semantics, SLOs, the error model (§31–§37).

**Sibling deep-dives this references:** `node.md` (on-node agent + immutable OS, the 4090 SKU, the 3.3 kW thermal system, tank/sensors/GPU power-state control), `connectivity-and-data.md` (outbound-only WireGuard overlay; caching, checkpoint/resume, regional caches, bandwidth-aware placement), `security-and-compliance.md` (attestation, isolation, key management, no-TEE honesty, trusted-host confidentiality class, legal/regulatory), `data-contracts.md` (NORMATIVE canonical schemas — job/tier/confidentiality/units/feasibility/usage), `financial-and-roadmap.md` (CapEx/OpEx, ABS covenants, phasing, concentration-risk register, seasonality risk), `operations.md` (observability/SLO/incident, testing/simulation, field operations), `system-design-overview.md` (the five principles and the headline economics).

---

## 0. Design principles — one preamble for the whole platform

The platform is a *direct* consequence of the overview's five realities. Everything in all four Parts follows from them, so they are restated once here as engineering constraints and not repeated per-Part:

| Principle | Platform consequence |
|---|---|
| **(1) Outbound-only** | The plane never dials a node. Nodes hold a persistent outbound stream (gRPC over the overlay, `connectivity-and-data.md`); all commands are *enqueued* and delivered on that stream. No service may assume an inbound address for a node. The yield router (Part B) likewise never talks to nodes directly — it emits an assignment to the scheduler. |
| **(2) Fail-safe-to-heater** | OTA, scheduler, and thermal coordinator emit *advisory* plans; the node agent's local safety logic can always override (`system-design-overview.md` §4.2). The plane must tolerate a node ignoring it. Comfort/safety is inviolable everywhere. |
| **(3) Thermally coupled** | The scheduler is a *joint* compute+heat optimizer; it cannot place work without a thermal window from the thermal coordinator. The utilization engine (Part C) is the math that produces those windows. |
| **(4) Measure-then-price** | A reliability score is computed continuously and is a first-class routing input, a tier gate, *and* a tape field. |
| **(5) Hybrid demand** | The scheduler consumes a single normalized `job` model regardless of source (marketplace vs own vs reserved vs direct), produced by the connectors / yield router (Part B). |

**Authoritative-data rule (load-bearing across all Parts):** the metering ledger (§7) is the single source of truth for GPU-hours and money. Telemetry (§2) is high-volume and *lossy/approximate*; it informs scoring and dashboards but is **never** the billing record. These two pipelines are deliberately separated — that separation is what lets the high-volume path be loose while the money path stays exact (§9), and it is why settlement reconciliation (§18) is a *money* check, not a telemetry check.

**Normative-schema rule:** `data-contracts.md` defines the canonical `superheat.job.v1`, `superheat.telemetry.v1`, `superheat.usage.v1`, the reliability score/tier ladder, the confidentiality axis, the units, and the `fits()` feasibility predicate. This document references those definitions and shows projections of them; it does **not** redefine them. Where any shape here disagrees with `data-contracts.md`, the contract wins.

---

# Part A — Control plane

The seven cloud services that turn scattered nodes into one fleet, with data models, interfaces, message flows, SLOs, and a build-vs-buy call for each, plus the scale architecture.

## 1. Registry & Inventory Service

The system of record for *what exists*. Low write rate, high read rate, strong consistency. **Buy:** managed Postgres (Supabase) — this is the same database that holds the ledger (§7), so registry FKs and settlement rows live in one transactional store.

### 1.1 Schema (Postgres)

```sql
-- An owner is whoever receives settlement: homeowner, commercial site, or Superheat itself.
CREATE TABLE owner (
  owner_id        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  legal_name      text NOT NULL,
  type            text NOT NULL CHECK (type IN ('homeowner','commercial','superheat')),
  payout_method   jsonb NOT NULL,           -- ACH/stripe-connect token, never raw bank data
  installer_id    uuid REFERENCES installer(installer_id),
  created_at      timestamptz NOT NULL DEFAULT now()
);

-- Standardized hardware SKUs (one cost-optimized 4090 SKU + commercial multi-GPU; see overview §3).
CREATE TABLE sku (
  sku_id          text PRIMARY KEY,          -- 'H1-4090-v2', 'COMM-4090x8-v1'
  gpu_model       text NOT NULL,             -- canonical token 'RTX4090' (data-contracts.md §1)
  gpu_count       int  NOT NULL,
  gpu_tdp_w       int  NOT NULL,             -- ~412
  cpu_cores       int  NOT NULL,
  ram_gb          int  NOT NULL,
  nvme_gb         int  NOT NULL,
  heater_kw       numeric NOT NULL,          -- 3.3 (H1); ~3.3 GPU-kW for the 8x4090 commercial SKU (NOT the disputed 33kW pitch figure — see node.md)
  usable_thermal_kwh numeric NOT NULL,       -- per-SKU tank battery capacity; ≈8.8 for 80-gal H1 (data-contracts.md §6); imported by the thermal coordinator (§6.1), NOT hardcoded
  form_factor     text NOT NULL CHECK (form_factor IN ('home','commercial'))
);

-- A node = one physical heater+host. The atomic fleet unit and the atomic asset unit.
CREATE TABLE node (
  node_id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  sku_id          text NOT NULL REFERENCES sku(sku_id),
  owner_id        uuid NOT NULL REFERENCES owner(owner_id),
  region          text NOT NULL,             -- 'us-east','us-mtn',... drives sharding + data POP (connectivity-and-data.md)
  geo             point,                     -- coarse lat/lng for climate/seasonality modeling only
  timezone        text NOT NULL,             -- needed by thermal coordinator pre-heat (§6)
  lifecycle_state text NOT NULL DEFAULT 'provisioned'
                  CHECK (lifecycle_state IN
                    ('provisioned','installed','attested','active',
                     'degraded','quarantined','maintenance','retired')),
  tranche_id      uuid REFERENCES tranche(tranche_id),  -- financing tag (§7)
  enrolled_at     timestamptz,               -- when first attested+active; tape "origination" date
  attestation_pk  bytea,                     -- node attestation public key (security-and-compliance.md)
  agent_version   text,                      -- last-reported node-agent version (node.md)
  os_slot         text CHECK (os_slot IN ('A','B')),
  created_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON node (region, lifecycle_state);
CREATE INDEX ON node (tranche_id);

-- Per-GPU row so a single 4090 inside an 8-GPU commercial node is independently rentable & meterable.
CREATE TABLE gpu (
  gpu_id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  node_id         uuid NOT NULL REFERENCES node(node_id),
  slot_index      int NOT NULL,              -- 0..gpu_count-1
  serial          text NOT NULL,             -- card serial; survives node, supports year-5 module swap
  vbios           text,
  state           text NOT NULL DEFAULT 'idle'
                  CHECK (state IN ('idle','renting','heating_only','faulted','disabled')),
  installed_at    timestamptz NOT NULL DEFAULT now(),
  retired_at      timestamptz                -- set on module swap; old serial kept for tape lineage
);
-- Only ONE LIVE card per slot. Retired rows (retired_at set) are kept for tape lineage and
-- must NOT block a fresh card going into the same slot at the year-5 module swap.
CREATE UNIQUE INDEX gpu_one_live_per_slot
  ON gpu (node_id, slot_index) WHERE retired_at IS NULL;

-- Lifecycle transitions are append-only — the registry's own audit trail.
CREATE TABLE node_lifecycle_event (
  event_id    bigserial PRIMARY KEY,
  node_id     uuid NOT NULL REFERENCES node(node_id),
  from_state  text, to_state text NOT NULL,
  reason      text, actor text,              -- 'scheduler','ota','operator:<id>','agent'
  at          timestamptz NOT NULL DEFAULT now()
);
```

### 1.2 Lifecycle state machine

```
provisioned → installed → attested → active ⇄ degraded
                                        │        │
                                        ▼        ▼
                                  maintenance  quarantined → (active | retired)
                                        │
                                        ▼
                                     retired
```

- `attested` requires a valid attestation from `security-and-compliance.md` (node is running signed Superheat software) before it may receive **paying** work — enforced by the scheduler (§5).
- `degraded` is set automatically by the reliability service (§4) when score crosses a tier boundary; `quarantined` by OTA halt (§3) or abuse detection.
- Only `active` nodes are *financeable*; the tape (§7) joins on `lifecycle_state='active'` plus a seasoning window.

### 1.3 Interface

Internal gRPC/REST `RegistryService`: `GetNode`, `ListNodes(filter)`, `UpdateLifecycle`, `AssignTranche`, `RegisterGpuSwap`. Reads are heavily cached (Redis, 30 s TTL) since the scheduler reads node metadata at high QPS. All writes go through Postgres transactions; lifecycle changes also append a `node_lifecycle_event`.

**Build vs buy:** *Buy* Postgres/Supabase; *build* the thin RegistryService API and the lifecycle state machine.

---

## 2. Telemetry & Observability Pipeline

High-volume, approximate, time-series. Drives dashboards, alerts, OTA health gates (§3), and reliability scoring (§4). **Never** the billing record.

### 2.1 Message flow

```
node-agent (node.md)
   │  batched, compressed telemetry frames every 10 s, flushed over the
   │  persistent outbound overlay stream (connectivity-and-data.md)
   ▼
[Edge Aggregator]  (regional; 1 per region/POP — critical at 1M nodes)
   │  - terminates node streams, validates attestation token
   │  - decimates: 10 s raw → 1 min rollups for cloud; keeps 10 s locally 24 h
   │  - back-pressure + buffer if cloud ingest is down (nodes never block on cloud)
   ▼
[Ingest Gateway]  (Kafka / Redpanda topic: telemetry.raw, keyed by node_id)
   │
   ├──► [TSDB writer]      → time-series store (metrics, 1-min rollups)
   ├──► [Scoring consumer] → reliability service (§4)
   └──► [Alert evaluator]  → on-call / dashboards
```

### 2.2 Telemetry frame (node → cloud, protobuf; JSON shown)

This is the **canonical `superheat.telemetry.v1` frame** defined in `data-contracts.md` §5 — not redefined here. Key canonical decisions the control plane relies on: GPUs are always a `gpus` **array** (length 1 for H1); GPU memory is `mem_used_mb` in **MB**; heat demand carries **both** `thermal.mode` (enum `none|realtime|storing|full`) and `thermal.heat_demand_w` in **watts**; network path is `net.path ∈ {direct,relay}`; and the aggregator stamps `cloud_degraded` on correlated outages (excluded from uptime scoring, §4). Illustrative projection:

```json
{
  "schema": "superheat.telemetry.v1",
  "node_id": "…", "agent_version": "2.4.1", "ts": "2026-06-18T12:00:10Z",
  "seq": 88231,                         // monotonic; node-attributable gaps => uptime loss (§4)
  "clock_skew_ms": 12,
  "gpus": [
    { "slot": 0, "model": "RTX4090", "util_pct": 97, "mem_used_mb": 21400,
      "power_w": 408, "power_cap_w": 450, "temp_c": 71, "xid_errors": 0 }
  ],
  "thermal": { "mode": "storing", "heat_demand_w": 1200,                 // mode enum + watts (both)
               "tank_temp_c": [58.2, 49.0], "storage_headroom_kwh": 4.1, // tank-as-battery (data-contracts.md §6)
               "comfort_ok": true },
  "board_power_w": 610,                                                    // whole-node wall draw
  "net": { "path": "direct", "uplink_mbps": 11.3, "downlink_mbps": 240,
           "rtt_ms": 28, "loss_pct": 0.1 },                              // direct vs relay (connectivity-and-data.md)
  "ckpt_age_s": 95,
  "cloud_degraded": false,              // aggregator-stamped; excluded from uptime (data-contracts.md §3)
  "sig": "ed25519:…"                    // node identity key (security-and-compliance.md)
}
```

See `data-contracts.md` §5 for the authoritative field list and types.

### 2.3 Storage & retention

| Tier | Store | Resolution | Retention | Purpose |
|---|---|---|---|---|
| Hot | TSDB (VictoriaMetrics / Timescale hypertable / Mimir) | 1 min | 30 days | dashboards, alerting, OTA gates |
| Warm | TSDB downsampled | 15 min | 13 months | seasonality (heat-demand), capacity planning |
| Cold | Object store (Parquet) | 1 min | 5 years | tape audit support, ML for forecasts (§6) |
| Edge | Aggregator local | 10 s | 24 h | live console, fine-grained incident replay |

> **Telemetry is not the tape.** Counters here are approximate (lossy under back-pressure). Authoritative GPU-hours come from signed, sequenced *job usage records* (§7), reconciled against telemetry only as a sanity check.

### 2.4 SLOs

- Telemetry ingest availability ≥ 99.9%; node→hot-tier visibility p95 < 60 s.
- Aggregator buffers ≥ 2 h of node telemetry during a cloud outage (no data loss for transient outages).
- Alert evaluation lag p99 < 30 s on the 1-min tier.

**Build vs buy:** *Buy* the TSDB, Kafka/Redpanda, Grafana. *Build* the edge aggregator (decimation + outbound-stream termination + attestation check — it is Superheat-specific) and the scoring/alert consumers.

---

## 3. OTA Update Orchestrator

Coordinates A/B image rollouts (defined in `node.md`) across the fleet. Never push to 100% at once; always gate on health; always able to halt and rollback.

### 3.1 Data model

```sql
CREATE TABLE ota_release (
  release_id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  artifact     text NOT NULL,                 -- content-addressed image ref (signed)
  version      text NOT NULL, channel text,   -- 'stable','beta'
  signature    bytea NOT NULL,                -- verified on-node before A/B swap
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ota_rollout (
  rollout_id   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  release_id   uuid NOT NULL REFERENCES ota_release(release_id),
  stage        text NOT NULL DEFAULT 'canary'
               CHECK (stage IN ('canary','cohort','fleet','halted','complete','rolled_back')),
  cohort_filter jsonb NOT NULL,               -- {region, sku_id, tier, percent}
  health_gate  jsonb NOT NULL,                -- see §3.3
  started_at   timestamptz, updated_at timestamptz
);

CREATE TABLE ota_node_status (
  rollout_id  uuid REFERENCES ota_rollout(rollout_id),
  node_id     uuid REFERENCES node(node_id),
  target_ver  text, result text              -- 'pending','applied','rolled_back','failed'
              CHECK (result IN ('pending','downloading','applied','rolled_back','failed','skipped')),
  applied_at  timestamptz,
  PRIMARY KEY (rollout_id, node_id)
);
```

### 3.2 Staged rollout flow

```
1. canary   1% of cohort, biased to LOW-reliability + idle (heating-only) nodes,
            never a non-interruptible reserved node.  Hold ≥ 6 h.
2. cohort   per-region waves, 5% → 25% → 50%; one region at a time.  Hold ≥ 2 h/wave.
3. fleet    remaining nodes, capped at N%/hour to bound blast radius.
Each transition is gate-checked (§3.3). Failure ⇒ auto-halt + auto-rollback of applied nodes.
```

A node updates only when the thermal coordinator (§6) reports a window and the node is not mid-job (or after checkpoint, `connectivity-and-data.md`). The orchestrator *enqueues* the target version; the agent applies the A/B swap on its own schedule and reports result. Because the OS does A/B + auto-rollback locally (principle 2), a bricked boot self-recovers even if the cloud never hears back.

### 3.3 Health gates (evaluated from telemetry §2)

Promote a stage only if, for updated nodes vs. a held-back control group:

- post-update **boot-success rate ≥ 99.5%** and `agent_version` reported within 30 min;
- **crash/watchdog-reboot rate** not >1.5× baseline;
- **GPU-fault / ECC** rate not elevated; **job failure rate** not elevated;
- **thermal-safety faults** = 0 new (hard stop — principle 2).

Breach ⇒ stage → `halted`, alert on-call, enqueue rollback-to-previous-slot for all `applied` nodes in the rollout, set affected nodes `quarantined` until clean.

### 3.4 SLOs

- Canary→fleet for a clean release completes < 14 days (deliberately slow; safety over speed).
- Auto-halt fires < 10 min after a gate breach.
- Zero customers left without hot water due to an update (principle 2) — measured: thermal-fault count attributable to OTA = 0.

**Build vs buy:** *Buy* the fleet/OTA substrate where possible (e.g., a Mender/Balena-style updater, or the chosen immutable-OS vendor's update server) for artifact distribution and A/B mechanics. *Build* the staged-rollout policy engine, health-gate evaluator, and the thermal/job-aware scheduling of when a node may apply — those are Superheat-specific and must integrate with §6 and `connectivity-and-data.md`.

---

## 4. Reliability / Uptime Scoring Service

Turns raw telemetry into a single per-node score and a tier. Consumed by the scheduler (§5), the yield router (Part B / §16), and the tape (§7) — seasoned high-reliability sub-pools are what financiers buy.

### 4.1 Inputs (all from telemetry §2, windowed)

| Component | Definition | Default window |
|---|---|---|
| `uptime` | fraction of expected heartbeats received (seq-gap based) | trailing 30 d |
| `thermal_avail` | fraction of time a usable thermal window existed (could-run hours ÷ total hours, from §6) | trailing 30 d |
| `net_quality` | normalized blend of upload mbps, RTT, loss, and direct-vs-relay ratio | trailing 7 d |
| `preempt_rate` | involuntary preemptions per 100 GPU-hours (heat-window loss, node drop, fault) | trailing 30 d |
| `fault_rate` | GPU/ECC/watchdog faults per 100 GPU-hours | trailing 30 d |

### 4.2 Score

The score and tier are the **canonical reliability formula and ladder** from `data-contracts.md` §3 and §2 — not redefined here. Positive weights sum to 1.0, so a perfect node reaches 100 and the top tier is actually reachable (the old local formula summed positive weights to 0.80, making `premium` mathematically unreachable):

```
# canonical (data-contracts.md §3): positive weights sum to 1.0 → perfect node = 100
base  = 0.45*uptime_frac + 0.25*thermal_avail + 0.20*net_quality + 0.10*(1 - preempt_penalty)
score = round( 100 * base * (1 - 0.30*fault_penalty) )      # faults scale down, max -30%

# EWMA-smooth so a single bad day doesn't whipsaw routing, and clamp:
score_t = clamp(0,100, α·score + (1-α)·score_{t-1})         # α = 0.3, daily

# canonical tier ladder (data-contracts.md §2), thresholds 85/70/50:
tier:  >=85 premium | >=70 standard | >=50 spot | else probation
```

- **`uptime_frac` excludes control-plane-down windows.** Heartbeat gaps from a *cloud/aggregator* outage are correlated across many nodes; the aggregator stamps `cloud_degraded` (telemetry §2.2) and those windows are excluded so a Superheat outage never penalizes a node's uptime. Only node-attributable gaps count.
- **Seasoning gate.** Regardless of score, a node with < 30 days of clean history is **`probation` and not tape-eligible** (`seasoned=false`). New nodes can earn (filler/backlog) but are not yet financeable collateral; the tape (§7) filters `seasoned=true`.

### 4.3 Stored output

```sql
CREATE TABLE node_reliability (
  node_id        uuid PRIMARY KEY REFERENCES node(node_id),
  score          numeric NOT NULL,
  tier           text NOT NULL,         -- premium|standard|spot|probation (data-contracts.md §2)
  uptime         numeric, thermal_avail numeric, net_quality numeric,
  preempt_rate   numeric, fault_rate numeric,
  seasoned       boolean NOT NULL DEFAULT false,
  computed_at    timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE node_reliability_history (   -- daily snapshots; feeds the tape's track record
  node_id uuid, score numeric, tier text, day date,
  PRIMARY KEY (node_id, day)
);
```

### 4.4 Consumption & SLO

- Scheduler/yield router read `tier` to gate work class via the canonical comparison rule (`data-contracts.md` §2): non-interruptible/contracted → `premium`/`standard`; spot/interruptible/backlog → `spot`/`probation` (overview §8.4). The engine's working uptime bands per tier are spelled out in §24.
- Tape (§7) joins `node_reliability_history` to prove a seasoned, high-reliability pool.
- SLO: score recomputed at least daily for every node; tier change propagates to scheduler cache < 5 min.

**Build vs buy:** *Build* (this is core IP and feeds underwriting). Runs as a §2 telemetry consumer + a daily batch job.

---

## 5. Scheduler / Orchestrator

Matches normalized demand to nodes subject to thermal window, reliability tier, interruptibility, and data-locality. It is the operational core. The *price* decision (which demand source pays most) belongs to the yield router in Part B; the scheduler decides *placement feasibility and assignment*.

### 5.1 The normalized job model

The scheduler consumes the **canonical `superheat.job.v1`** object defined in `data-contracts.md` §4 (produced by the connectors / yield router in Part B). The flat shape below is a **PROJECTION** of that canonical object — the fields the scheduler reads, mapped onto the nested canonical names. See `data-contracts.md` §4 for the authoritative schema; the canonical fields win. (Part B / §13 specifies how connectors normalize each marketplace into this object; Part D / §31 specifies the external renter projection of it.)

```jsonc
// PROJECTION of superheat.job.v1 (data-contracts.md §4) — canonical names/units
{
  "schema": "superheat.job.v1",
  "job_id": "…",
  "source": "vast",                        // enum: vast|runpod|salad|ionet|akash|direct|reserved|backlog
  "gpu_count": 1, "gpu_model": "RTX4090", "min_vram_mb": 24000,   // canonical token + MB
  "interruptible": true,
  "reliability_tier_required": "spot",     // §2 ladder; reserved/contracted set standard+
  "expected_duration_s": 5400,             // SECONDS
  "min_thermal_window_s": 1800,            // SECONDS of contiguous runnable time to start
  "data_locality": ["us-east"],            // where weights/datasets are cached (connectivity-and-data.md)
  "net_price_usd_per_gpu_hr": 0.30,        // gross*(1-fee); the number the yield router compares
  "max_start_latency_s": 120,
  "confidentiality_required": "standard"   // standard|trusted_host (data-contracts.md §2.2)
}
```

### 5.2 Matching algorithm (sketch)

```
function place(job):
  cand = registry.nodes(state='active', attested=true,
                        gpu_model = job.gpu_model, free_gpus >= job.gpu_count)

  # HARD filter: the canonical feasibility predicate (data-contracts.md §7).
  # fits() encapsulates tier ladder, confidentiality, gpu_count, and the
  # interruptible-vs-non-interruptible thermal-window-fit rule (incl. T_GRACE_S=30).
  cand = [n for n in cand
          if fits(job, n, thermal_coordinator.window(n).usable_seconds)]

  # SCORE feasible candidates (higher = better placement)
  for n in cand:
    s =  α * n.reliability.score        # α = 1.0   (0..100 reliability score)
       + β * data_locality_match(job, n) # β = 25    (1.0 in-region cache hit, else 0)
       + γ * thermal_fit(job, n)         # γ = 15    (window length vs job; prefer banking heat now)
       + δ * net_quality_for(job, n)     # δ = 10    (data-heavy job → better uplink, 0..1)
       - ε * preemption_risk(job, n)     # ε = 40    (window ending soon + non-interruptible, 0..1)
       - ζ * fragmentation_penalty(n)    # ζ = 20    (avoid stranding GPUs on commercial nodes, 0..1)
  pick argmax s ; if none → REQUEUE (backlog) or REJECT (deadline jobs)
  reserve gpu rows (state→renting) in a txn; emit placement to node via outbound stream
```

- **Example weights** above are illustrative starting values (tuned per region from outcomes); reliability dominates, preemption risk is the strongest penalty. `MIN_SLICE` is **not** a scheduler constant — the minimum runnable window is `job.min_thermal_window_s` for interruptible jobs and the full `expected_duration_s` for non-interruptible jobs, both enforced inside the canonical `fits()` (`data-contracts.md` §7, with `T_GRACE_S = 30 s`).
- **Anti-thrash / hysteresis.** To avoid churning placements on noisy scores or flickering windows, a feasible incumbent assignment is only displaced if a challenger's score beats it by a margin (e.g. `> 10%`) sustained over a debounce interval (e.g. `≥ 60 s`); tier-cache updates (§4.4) are EWMA-smoothed before they can demote a running node. A job is not re-placed unless it was actually preempted. (The utilization engine applies the same hysteresis on yield preemption — §29 step 5.)

Premium/reserved jobs are placed first (priority queue); the interruptible **own-backlog filler** (§15, overview §8.2) is the lowest priority and exists precisely to consume any window no paying job claimed — so windows are never wasted.

### 5.3 Job lifecycle

```
QUEUED → MATCHED → DISPATCHED → STARTING → RUNNING ─┬─► COMPLETED
                                    ▲                ├─► PREEMPTED → (checkpoint, connectivity-and-data.md) → QUEUED
                                    └── resume ──────┘   (resumes elsewhere)
                                                     └─► FAILED → (retry policy) → QUEUED | DEAD
```

Every transition emits a **job usage record** (§7) carrying signed start/stop and metered GPU-seconds — this is the billing primitive, not telemetry. (The external renter-facing projection of this state machine — `QUEUED/SCHEDULED/RUNNING/PREEMPTED/RESUMING/SUCCEEDED/...` — is in Part D / §34.)

```sql
CREATE TABLE job (             -- persisted projection of superheat.job.v1 (data-contracts.md §4)
  job_id        uuid PRIMARY KEY,
  source        text NOT NULL, tenant_ref text,           -- source ∈ full enum (data-contracts.md §4)
  gpu_count     int, interruptible boolean,
  reliability_tier_required text,                          -- premium|standard|spot|probation
  state         text NOT NULL,
  net_price_usd_per_gpu_hr numeric,
  created_at    timestamptz, updated_at timestamptz
);
CREATE TABLE job_assignment (
  assignment_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id        uuid REFERENCES job(job_id),
  gpu_id        uuid REFERENCES gpu(gpu_id),
  node_id       uuid REFERENCES node(node_id),
  started_at    timestamptz, ended_at timestamptz,
  end_reason    text          -- 'completed','preempted_thermal','preempted_yield','fault'
);
```

### 5.4 SLOs

- Placement decision p95 < 2 s for available capacity.
- Wrongful placement of a non-interruptible job onto a window that ends early (causing involuntary preemption) < 0.5% of premium placements.
- Idle-window leak (a usable thermal window with no job for >5 min when backlog exists) < 1%.

**Build vs buy:** *Build* — the scheduler is the moat (overview §6). Stateless workers behind a queue; node-assignment writes are transactional in Postgres; hot node-state in Redis.

---

## 6. Thermal Coordinator

Per-site model that treats the **tank as a battery** and tells the scheduler when each node can run compute and for how long. The full physics/forecast model lives in Part C (§20–§23); this section specifies the *control-plane interface* the coordinator exposes.

### 6.1 Per-site tank-as-battery model

```json
{
  "node_id": "…",
  "tank": { "usable_thermal_kwh": 8.8,        // per-SKU registry field; ≈8.8 kWh for 80-gal H1 (data-contracts.md §6)
            "temp_c": 58.2, "min_comfort_c": 49,
            "max_safe_c": 70, "loss_w": 90 },
  "stored_kwh": 7.3,                          // usable heat banked now
  "headroom_kwh": 4.1,                        // how much GPU heat we can still absorb
  "comfort_constraint": { "guaranteed_hot_water": true,
                          "schedule": "ical-or-learned-occupancy" },
  "gpu_heat_w": 408                           // per active GPU, into the loop
}
```

`usable_thermal_kwh` is **not hardcoded** — it is the per-SKU `usable_thermal_kwh` registry field (`data-contracts.md` §6, §9), imported by the coordinator; the canonical value for the standard 80-gal H1 tank is `E_store ≈ 8.8 kWh` (commercial/larger-tank SKUs carry their own). State of charge: `SoC = stored_kwh / usable_thermal_kwh`. The GPU can run as long as **headroom > GPU heat × slice** OR the banked heat will be consumed by forecasted demand before the tank overheats. (The full charge/discharge dynamics and the thermal-window computation are §20.2 and §22.)

### 6.2 Heat-demand forecast & pre-heat

- Forecasts next-24 h hot-water/heat demand per site from learned occupancy + weather + season (cold vs warm climate; summer DHW-only — overview §12), using warm-tier telemetry (§2.3) and the node's `timezone`. The full forecaster is §20.4.
- **Pre-heat scheduling:** if a high-demand window is forecast, the coordinator *requests* the scheduler to run compute ahead of it to bank heat — converting a future heating cost into compute revenue now. This is the key utilization unlock (overview §8.1). The pre-heat feasibility math (and the storage guard) is §20.3.

### 6.3 Interface to the scheduler

```protobuf
service ThermalCoordinator {
  // Can this node run compute right now, and for how long?
  rpc GetWindow(NodeId) returns (ThermalWindow);
  rpc StreamWindows(RegionId) returns (stream ThermalWindowUpdate); // scheduler subscribes
}
message ThermalWindow {
  string node_id = 1;
  bool   can_run_now = 2;
  int64  usable_seconds = 3;      // until headroom exhausted OR comfort risk
  double soc = 4;                 // 0..1
  bool   preheat_requested = 5;   // coordinator WANTS compute now to bank heat
  double max_continuous_w = 6;    // throttle ceiling (may be < full TDP)
}
```

The coordinator also emits, per node per interval, a **runnable-minutes budget** and a **mode hint** (`discharge` / `charge` / `preheat` / `hold`, §20.3); the scheduler consumes that and never computes thermal feasibility itself.

**Hard rule (principle 2):** comfort/safety constraints are inviolable. If honoring a running job would risk hot-water comfort or exceed `max_safe_c`, the coordinator marks the window closed; the node agent enforces locally regardless of what the cloud said. The scheduler treats `ThermalWindow` as a hard filter (§5.2).

### 6.4 SLOs

- Window prediction error (predicted usable_seconds vs actual) p50 within ±15%.
- Comfort-violation rate attributable to compute = 0 (hard).
- Pre-heat decisions improve thermal_avail-weighted utilization (tracked against the Part C targets: 60% util × 70% overlap — §23).

**Build vs buy:** *Build* — joint compute+heat optimization is core IP. Per-node lightweight model in the cloud (and a safety twin on-node, `node.md`).

---

## 7. Metering & Billing Ledger — the Securitization Tape

The authoritative GPU-hours/$ record. Per-unit accurate, auditable, append-only, reconciled. **This ledger *is* the static-pool tape** from `financial-and-roadmap.md`. Lives in the same Postgres as the registry (§1) so settlement and node identity are transactional together. Settlement reconciliation against marketplace payouts is Part B / §18.

### 7.1 The billing primitive: signed usage records

Metering is **not** derived from telemetry. The node agent emits a **signed, sequenced usage record** at job stop / checkpoint / heartbeat-roll, and the cloud counter-signs:

```json
{
  "usage_id": "…", "job_id": "…", "assignment_id": "…",
  "node_id": "…", "gpu_slot": 0, "owner_id": "…", "tranche_id": "…",  // per-GPU: node_id + gpu_slot (data-contracts.md §8)
  "source": "vast", "tenant_ref": "…",
  "start": "2026-06-18T10:00:00Z", "end": "2026-06-18T11:30:00Z",     // NEVER spans a billing-period boundary; split at boundary
  "gpu_seconds": 5400,
  "realized_usd_per_gpu_hr": 0.41,
  "gross_usd": 0.615,
  "node_seq": 88231,                 // ties to telemetry seq for reconciliation
  "node_sig": "…",                   // agent-signed (security-and-compliance.md key)
  "cloud_sig": "…"                   // counter-signed on ingest
}
```

This is the **canonical `superheat.usage.v1`** primitive (`data-contracts.md` §8): metered **per-GPU** (`gpu_slot`, not per-node) and a record **never spans a billing-period boundary** — long jobs emit one record per period, split at the boundary, which keeps monthly `utilization_pct` correct. Double-signing makes each GPU-hour non-repudiable by both node and platform — exactly the auditability an ABS underwriter needs.

### 7.2 Ledger tables (append-only)

```sql
CREATE TABLE usage_record (              -- immutable; the metered truth (superheat.usage.v1, data-contracts.md §8)
  usage_id      uuid PRIMARY KEY,
  job_id uuid, assignment_id uuid,
  node_id uuid REFERENCES node(node_id),
  gpu_id uuid NOT NULL REFERENCES gpu(gpu_id),  -- the exact physical card row (survives slot reuse / year-5 swap)
  gpu_slot int NOT NULL,                 -- = gpu.slot_index at metering time; per-GPU, NOT per-node
  owner_id uuid REFERENCES owner(owner_id),
  tranche_id uuid REFERENCES tranche(tranche_id),
  source text, tenant_ref text,
  start_ts timestamptz, end_ts timestamptz,   -- never spans a billing-period boundary (split at boundary)
  gpu_seconds bigint NOT NULL,
  realized_usd_per_gpu_hr numeric NOT NULL,
  gross_usd numeric NOT NULL,
  node_sig bytea, cloud_sig bytea,
  recorded_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON usage_record (node_id, gpu_slot, start_ts);
CREATE INDEX ON usage_record (tranche_id, start_ts);

CREATE TABLE settlement (               -- periodic owner payout + platform take
  settlement_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id uuid, node_id uuid, period tstzrange,
  gross_usd numeric, platform_fee_usd numeric,   -- 25% take (overview §3)
  electricity_offset_usd numeric,                -- embedded heating cost note
  net_owner_usd numeric, status text,            -- 'pending','paid','failed'
  marketplace_payout_ref text,                   -- for reconciliation (§18)
  created_at timestamptz DEFAULT now()
);

CREATE TABLE tranche (                   -- financing pool a node is assigned to
  tranche_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text, warehouse_facility text,
  vintage date, status text             -- 'warehouse','issued','retired'
);
```

### 7.3 Static-pool tape (the deliverable to capital markets)

A materialized view, refreshed per reporting period, one row per node per period — the fields an underwriter rates (`financial-and-roadmap.md`, ABS one-pager):

**JOIN correctness (was a bug).** The prior view joined `gpu` and `usage_record` to `node` *independently*, so each usage record was multiplied by every GPU on the node — a node with 8 GPUs reported 8× its real revenue and GPU-hours. The fix joins `usage_record` to `gpu` on **both** `node_id` AND `gpu_slot` (usage records are per-GPU, `data-contracts.md` §8), so each record maps to exactly one physical card. We aggregate per GPU first, then roll up to the node row, and because a usage record never spans a period boundary (`data-contracts.md` §8) every record falls cleanly in one `period`.

```sql
CREATE MATERIALIZED VIEW securitization_tape AS
WITH gpu_period AS (   -- one row per (node, gpu, month): aggregate earnings PER GPU
  SELECT
    u.node_id,
    u.gpu_slot,
    date_trunc('month', u.start_ts)  AS period,
    sum(u.gpu_seconds)               AS earning_gpu_seconds,
    sum(u.gross_usd)                 AS gross_usd
  FROM usage_record u
  -- per-GPU join on BOTH keys (fixes the Cartesian multiply):
  JOIN gpu g ON g.node_id = u.node_id AND g.slot_index = u.gpu_slot
  GROUP BY u.node_id, u.gpu_slot, date_trunc('month', u.start_ts)
)
SELECT
  n.node_id, n.tranche_id, n.sku_id, n.region,
  n.enrolled_at                         AS origination_date,
  gp.period,
  count(distinct gp.gpu_slot)           AS earning_gpu_count,   -- GPUs that actually earned this period
  s.gpu_count                           AS installed_gpu_count, -- physical GPUs on the SKU
  sum(gp.earning_gpu_seconds)/3600.0    AS gpu_hours,
  sum(gp.gross_usd)                     AS gross_revenue_usd,
  sum(gp.gross_usd) * 0.25              AS platform_fee_usd,
  rel.score                             AS reliability_score,
  rel.tier                              AS reliability_tier,
  rel.uptime, rel.thermal_avail, rel.seasoned,
  sum(gp.gross_usd) / nullif(sum(gp.earning_gpu_seconds)/3600.0,0) AS realized_usd_per_gpu_hr,
  -- utilization = earning GPU-seconds ÷ total GPU-seconds AVAILABLE in the period
  -- (installed GPUs × seconds in the month). No Cartesian inflation; capped at 1.0.
  (sum(gp.earning_gpu_seconds))
     / (s.gpu_count * extract(epoch FROM (
          (gp.period + interval '1 month') - gp.period)))       AS utilization_pct
FROM gpu_period gp
JOIN node n               ON n.node_id = gp.node_id
JOIN sku  s               ON s.sku_id  = n.sku_id
JOIN node_reliability rel ON rel.node_id = n.node_id
WHERE n.lifecycle_state = 'active' AND rel.seasoned = true
GROUP BY n.node_id, n.tranche_id, n.sku_id, n.region,
         n.enrolled_at, gp.period, s.gpu_count,
         rel.score, rel.tier, rel.uptime, rel.thermal_avail, rel.seasoned;
```

This yields: origination date, vintage/tranche, correct per-node GPU-hours and gross revenue, realized $/GPU-hr, a true utilization % (earning GPU-seconds over the installed GPUs' available GPU-seconds in the period), and the reliability track record — i.e., a credit file an underwriter can rate (`financial-and-roadmap.md` Phase 1).

### 7.4 Reconciliation (auditability)

Reconciliation runs every period; discrepancies open an audit ticket and block settlement for the affected node. The marketplace-payout side of this loop (currency normalization, fee truth-up, the reconciliation pipeline diagram) is detailed in Part B / §18.

**The ±tolerance is a money check, not a telemetry check.** Per `data-contracts.md` §8, the authoritative reconciliation is **signed `usage_record`s vs marketplace settlement** (`net_usd`/`gross_usd` summed per marketplace must reconcile to that marketplace's payout statement). The ±tolerance applies *there* — to money. It does **not** apply to telemetry: telemetry (§2) is lossy and exists for health/scoring only, never for billing, so it is never reconciled against the ledger on a tolerance. (This corrects the prior "reconcile lossy telemetry within ±2%" claim.)

1. **Usage ↔ marketplace payouts (the money reconciliation):** `settlement.marketplace_payout_ref` matches what vast.ai/RunPod/etc. actually paid (from the Part B connectors). Our internal `gross_usd`/`net_usd` must reconcile with external payout statements within the configured ± tolerance — this catches connector/price drift and proves the tape to investors.
2. **Usage ↔ telemetry (a sanity signal, NOT a tolerance gate):** signed usage GPU-seconds may be *compared* against telemetry-derived busy-seconds only as a coarse anomaly signal (e.g., a node billing while telemetry shows it idle flags meter tampering or agent bugs). A divergence here triggers investigation — it never adjusts the ledger and is not subject to the money ± tolerance, because telemetry is approximate by design.

Owner settlement = `gross − platform_fee (25%)`, paid per period; platform take and electricity-offset notes are recorded for the four-party split (overview / pitch slide 9). The per-tranche split (owner payout, Superheat servicing fee, platform take) is detailed in §18.

### 7.5 SLOs

- Per-node metering accuracy: 100% of paid GPU-hours backed by a double-signed `usage_record`.
- Reconciliation drift > 2% triggers an audit within one period.
- Tape refresh: monthly close completes < 24 h after period end, fully reproducible from the append-only `usage_record` table (re-runnable view = audit guarantee).

**Build vs buy:** *Buy* Postgres/Supabase + Stripe Connect (or similar) for payout rails. *Build* the metering pipeline, double-signing, the tape view, and reconciliation — this is the asset-origination moat and must be correct from day one.

---

## 8. End-to-end request / job sequence

```
Renter/Marketplace        Yield Router            Scheduler        Thermal Coord.     Node Agent          Ledger
   (Part B, §16)                                    (§5)              (§6)              (node.md)           (§7)
        │  job request          │                    │                 │                  │                 │
        │──────────────────────►│                    │                 │                  │                 │
        │                       │ pick highest-$ src │                 │                  │                 │
        │                       │ + normalize job    │                 │                  │                 │
        │                       │───────────────────►│ place(job)      │                  │                 │
        │                       │                    │── GetWindow ───►│                  │                 │
        │                       │                    │◄── window ──────│                  │                 │
        │                       │   (hard-filter by window, tier,      │                  │                 │
        │                       │    locality; score candidates)       │                  │                 │
        │                       │                    │── DISPATCH (over outbound stream) ─►│                 │
        │                       │                    │                 │   pull image/    │                 │
        │                       │                    │                 │   weights from   │                 │
        │                       │                    │                 │   regional cache │                 │
        │                       │                    │                 │ (connectivity-…) │                 │
        │                       │                    │◄──── STARTING / RUNNING ────────────│                 │
        │◄── access via overlay (connectivity-and-data.md) ──────────────────────────────│                 │
        │                       │                    │                 │   ...running...  │                 │
        │                       │                    │   thermal window closing /          │                 │
        │                       │                    │◄── window update (preempt) ─────────│                 │
        │                       │                    │── PREEMPT ─────────────────────────►│ checkpoint      │
        │                       │                    │                 │                  │ (connectivity-…)│
        │                       │                    │                 │                  │── usage rec ───►│ sign+
        │                       │                    │   REQUEUE → resume elsewhere        │                 │ counter-sign
        │                       │                    │                 │                  │── usage rec ───►│ (§7)
        │                       │                    │                 │  job COMPLETED   │                 │
        │                       │                    │                 │                  │   settlement →  │ owner payout
```

---

## 9. Scale to ~1GW / up to ~1M nodes — architectural implications

The target is up to ~1M nodes (~1GW at Blackwell-class ≈1kW; pitch slide 11). That is the dominant force on the architecture:

- **Edge aggregation is mandatory.** 1M nodes × a 10 s telemetry frame ≈ 100K frames/s. Terminating that centrally is wasteful and fragile. Regional **edge aggregators** (§2.1) terminate node streams, decimate 10 s→1 min, buffer through cloud outages, and hold a **read-only** cache of latency-sensitive data near the nodes. Cloud ingests rollups, not raw.
- **Edge caches reads; reservations stay central and transactional.** This reconciles the edge role with the scheduler (§5). Edge aggregators may cache READ-ONLY thermal-window snapshots and reliability/scoring data so feasibility scoring (§5.2) runs with low latency near the nodes. But **GPU reservations are NEVER made at the edge** — the actual `gpu` row reservation (`state→renting`) is a single **central, transactional** Postgres write (§5.2, §1.3). This is the only thing that prevents two edges from double-booking the same GPU; a stale edge cache can at worst cause a placement attempt that the central transaction rejects (the GPU is taken), never a double-booking.
- **Shard by region.** `node.region` is the shard key for telemetry, scheduling, and the thermal coordinator. A node only ever talks to its regional plane; cross-region is rare (a renter wanting capacity elsewhere). Registry + ledger Postgres shards by region/tranche; the tape view fans in per region at period close.
- **Eventual consistency where it's safe; strong where it's money.** Telemetry, reliability scores, and scheduler node-state caches are **eventually consistent** (Redis, seconds of staleness is fine). Registry lifecycle, job assignment (no double-booking a GPU), and the ledger are **strongly consistent** (Postgres transactions). The deliberate telemetry-vs-ledger split (§0) is what lets the high-volume path be loose while the money path stays exact.
- **Backpressure & node autonomy.** Because nodes are outbound-only and fail-safe-to-heater, a regional plane outage degrades gracefully: nodes keep heating, keep running already-dispatched jobs, buffer usage records, and reconcile on reconnect. No node depends on the cloud to stay safe.
- **Stateless control workers.** Scheduler, scoring, OTA-policy, and reconciliation are stateless horizontal workers behind queues; state lives in Postgres/Redis/TSDB. Scale = add workers per region.
- **Tranche-aware everything.** At fleet scale the *financing* dimension (tranche) is as load-bearing as region: usage records, settlement, and the tape all index by `tranche_id` so issuance can pull a clean static pool without a full-fleet scan.

---

## 10. Control-plane build-vs-buy summary

| Service | Buy | Build |
|---|---|---|
| Registry & inventory (§1) | Postgres / Supabase | RegistryService API, lifecycle FSM |
| Telemetry (§2) | TSDB, Kafka/Redpanda, Grafana | Edge aggregator, scoring/alert consumers |
| OTA (§3) | OS-vendor / Mender-Balena artifact + A/B substrate | Staged-rollout + health-gate + thermal-aware apply policy |
| Reliability scoring (§4) | — | Entire service (core IP) |
| Scheduler (§5) | Redis, queue | Entire scheduler (the moat) |
| Thermal coordinator (§6) | — | Entire service + on-node safety twin (core IP) |
| Metering & ledger (§7) | Postgres, Stripe Connect payout rails | Metering, double-signing, tape view, reconciliation (the origination moat) |

**One-line takeaway:** buy the boring substrate (database, time-series, queues, OTA transport, payout rails); build the four things that are the business — the scheduler, the thermal coordinator, the yield router (Part B), and the metering ledger that becomes the securitization tape.

---

# Part B — Demand & yield routing

This Part answers Q4 ("auto-dispatch + integrate marketplaces") and the demand half of Q5 ("own network"). It is deliberately **hybrid, not a fork**: we attach to existing marketplaces *first* for instant fill, build our own/direct demand *in parallel*, and route every node-hour to the highest-yield source at that moment. Its four sub-parts are: marketplace connectors + the normalized job (§13), own demand + the interruptible filler (§15), the yield router (§16), and settlement reconciliation back into the metering ledger (§18).

## 11. Mental model

A Superheat node is a single RTX 4090 that can run compute **only inside a thermal window** (the tank can absorb/store the heat — see Part C). The demand layer's job is to keep that window full of the *highest-paying* work that the window's length, the node's reliability tier, and the concentration guard permit.

```
   SOURCES (heterogeneous APIs, fees, payout latency)
   ┌────────────┬─────────┬───────┬────────┬───────┐   ┌──────────────┬──────────────┐
   │  vast.ai   │ RunPod  │ Salad │ io.net │ Akash │   │ direct/API + │ interruptible │
   │            │         │       │        │       │   │ reserved     │ batch backlog │
   └─────┬──────┴────┬────┴───┬───┴───┬────┴───┬───┘   └──────┬───────┴──────┬───────┘
         │ connector (one per source) → normalize → internal Job             │
         └──────────────────────────────┬──────────────────────────────────-┘
                                         ▼
                            ┌─────────────────────────┐   inputs from Part A:
                            │      YIELD ROUTER        │ ◄ thermal window (§6), reliability tier (§4),
                            │  pick max E[$/GPU-hr]    │   node network score, current bids
                            │  s.t. thermal/relia/conc │
                            └────────────┬────────────┘
                                         ▼  Job assignment → scheduler (§5)
                                         ▼  delivery telemetry → metering ledger (§7)
                                         ▼  marketplace payouts → settlement reconciliation (§18)
```

The router does **not** talk to nodes directly. It emits a normalized `JobAssignment` to the scheduler (§5), which enforces local thermal/safety limits (local safety always wins — `system-design-overview.md` §4.2).

## 12. Marketplace connectors — why this order

| # | Source | Why this rank | Integration model | Job arrival | Settlement / payout | Take rate | Interruptible? | Verification req. |
|---|---|---|---|---|---|---|---|---|
| 1 | **vast.ai** | Deepest 4090 demand; host model built for heterogeneous/consumer HW; 4090 listed $0.38–0.41, lows ~$0.17 (`GPU_Pricing_Research_4090_H100.md`) | Host runs `vastai` host daemon on the node; we list the GPU at an ask price; renter rents it | Renter picks our listing → daemon pulls renter container | Marketplace credits host balance; we withdraw on a cycle (days) | ~25–28% effective host commission | Yes — "interruptible" (bid) and "on-demand" listing types | DLPerf benchmark, reliability score, host verification badge |
| 2 | **RunPod** | Large 4090 demand; Community Cloud is the consumer-host tier ($0.34 in Jun 2026) | Host joins as a Community Cloud machine; agent registers capacity | Pod scheduled onto our machine by RunPod | Per-second billing → host earnings ledger; payout cycle | ~25–35% (Community vs Secure split) | Community pods are pre-emptible; Secure are not | Machine vetting, uptime SLA tier |
| 3 | **Salad** | Purpose-built for **distributed/consumer** nodes; lowest floor ($0.18) but zero-friction fill | Lightweight container host agent (SaladCloud) | Org's container replicas auto-placed across the consumer fleet | Earnings accrue; withdraw threshold | ~indirect (Salad keeps spread between org price and host pay) | Fully pre-emptible by design (replicas churn) | Node passes hardware/bandwidth check |
| 4 | **io.net** | Aggregator/DePIN; clusters consumer GPUs into on-demand pools | `io worker` binary joins a device cluster | Cluster jobs dispatched to worker | Token + fiat settlement; on-chain payout latency | Protocol fee (~ low-double-digit %) | Yes (cluster jobs are batch-like) | Device verification, proof-of-compute |
| 5 | **Akash** | Decentralized lease marketplace; bid/reverse-auction | Provider runs Akash provider stack; we bid on deployments | Tenant accepts our bid → lease created | On-chain escrow → AKT/USDC; settled per block, claimable | Low protocol take; bears token volatility | Leases can be closed; treat as interruptible | Provider attributes/audit, manifest match |

**Build order rationale** (see `financial-and-roadmap.md` Phase A): vast.ai is the single fastest path to revenue and the deepest 4090 pool, so it ships first and alone in Phase A. RunPod + Salad follow because they explicitly serve consumer/distributed hosts (lowest integration risk). io.net + Akash are DePIN/on-chain and add payout-latency and token-volatility complexity, so they are last and gated behind the settlement reconciliation work in §18.

## 13. The connector abstraction & the normalized job

### 13.1 The connector abstraction

Every source is wrapped in a connector that implements one interface. The router and scheduler never see source-specific concepts — only the normalized `Job`. This is what makes adding a sixth marketplace a contained change.

```python
class MarketplaceConnector(Protocol):
    source_id: str            # vast|runpod|salad|ionet|akash|direct|reserved|backlog (data-contracts.md §4)
    settlement: SettlementProfile  # fee %, payout latency, currency, dispute window

    # --- price discovery (read) ---
    def quote(self, node: NodeProfile, window: ThermalWindow) -> SourceQuote:
        """Return expected price + fill probability + preemption risk for THIS node,
        right now, given how long the thermal window is. May call the source API,
        a cached price feed, or a learned model (see §16.2). Cheap & idempotent."""

    # --- listing lifecycle (write) ---
    def list(self, node: NodeProfile, ask: AskPolicy) -> ListingHandle:
        """Publish/refresh our availability on the source (e.g. set vast.ai ask price,
        register a RunPod machine slot, declare a Salad capacity unit)."""
    def unlist(self, handle: ListingHandle) -> None: ...

    # --- job intake → normalize ---
    def poll(self) -> list[Job]:
        """Pull newly-assigned work from the source and emit normalized Jobs.
        Push-capable sources (webhooks) feed the same normalization path."""
    def normalize(self, raw: RawSourceJob) -> Job: ...

    # --- runtime control ---
    def on_assigned(self, job: Job, node_id: str) -> None: ...
    def preempt(self, job: Job, reason: PreemptReason) -> PreemptReceipt:
        """Cleanly stop a source's job (checkpoint via connectivity-and-data.md, release the slot)."""
    def heartbeat(self, job: Job) -> SourceJobState: ...

    # --- settlement (§18) ---
    def pull_settlement(self, since: Timestamp) -> list[PayoutRecord]:
        """Fetch realized payouts to reconcile against the metering ledger."""
```

`SettlementProfile` carries the economics the router needs at decision time:

```python
@dataclass
class SettlementProfile:
    fee_pct: float            # 0.27 for vast, 0.30 for runpod-community, etc.
    payout_latency_days: float# vast ~2-5, akash ~minutes-on-chain but claim batched
    currency: str             # "USD" | "AKT" | "IO" ...
    fx_volatility: float      # 0 for fiat; >0 for token sources (haircut in router)
    dispute_window_days: int  # how long a payout can be clawed back
```

### 13.2 The normalized internal Job schema

The normalized job is **THE canonical `superheat.job.v1`** defined in `data-contracts.md` §4 — not a doc-local shape. Connectors translate each marketplace's native job into exactly that schema; the router and scheduler consume it (the scheduler consumes a **projection** of it — §5.1, `data-contracts.md` §9). This section only adds the connector-side fields the router needs at decision time that ride alongside the canonical object; **canonical field names, enums, and units (seconds for durations) are authoritative in `data-contracts.md`** and are not restated here.

Key alignment points the connector MUST honor (per `data-contracts.md` §4):
- `source ∈ {vast,runpod,salad,ionet,akash,direct,reserved,backlog}` (full enum).
- `compute.gpu_model: "RTX4090"` (canonical token, §1), `compute.expected_duration_s` in **seconds** (null = open-ended).
- `scheduling.reliability_tier_required ∈ {probation,spot,standard,premium}` ONLY. `trusted_host` is **not** a reliability tier — it is a confidentiality axis (`trust.confidentiality_required ∈ {standard,trusted_host}`, `data-contracts.md` §2.2).
- `scheduling.min_thermal_window_s` in **seconds**; `scheduling.interruptible` gates the feasibility predicate (`fits()`, §16.3 / `data-contracts.md` §7).
- `economics.net_price_usd_per_gpu_hr = gross_price_usd_per_gpu_hr · (1 − platform_fee_frac)` — the router only ever compares **net**.

Connector-side decision-time fields (not on the canonical wire object; held by the connector's `SourceQuote`/`SettlementProfile`, see §13.1): `fx_haircut_pct` (token sources), `payout_latency_days`, `preempt_penalty_usd_per_gpu_hr`. Marketplace-specific raw fields are kept opaque for settlement/debug only and never leak into routing logic.

Three derived invariants the connector MUST populate:
1. `economics.net_price_usd_per_gpu_hr = gross · (1 − platform_fee_frac)`, further reduced by the connector's `fx_haircut_pct` for token sources — the router only ever compares **net**.
2. `scheduling.reliability_tier_required` — non-interruptible/premium jobs are illegal on lower-tier nodes; comparison is the single `tier_order` rule (`data-contracts.md` §2). A `trust.confidentiality_required == "trusted_host"` job additionally requires a `trusted_host` node (orthogonal axis).
3. `scheduling.min_thermal_window_s` — jobs that can't fit the node's remaining window are filtered by the canonical `fits()` predicate (`data-contracts.md` §7) before the router scores them.

## 14. Own demand — direct API + reserved contracts

Marketplaces give instant fill but cost 25–35% in fees and can become a concentration risk. Own demand is how we (a) capture that fee back as margin and (b) guarantee there is *always* something to run.

- **Direct API** (`tenant_class: "direct"`): customers submit jobs straight to Superheat's API (the external surface is Part D). No marketplace fee, so `platform_fee_frac = 0` and the **same** $0.40 gross becomes ~$0.40 net instead of ~$0.29. This is the highest-margin spot demand and is the connector with `source = "direct"` — it implements the exact same `MarketplaceConnector` interface (§13.1), so the router treats it as just another source (one that happens to have zero fee).
- **Reserved-instance contracts** (`tenant_class: "reserved"`): a customer pre-commits to N GPU-hours/month at a contracted floor price (e.g. $0.49/GPU-hr reserved, per `GPU_Pricing_Research_4090_H100.md` reserved median). These create **reserved-floor commitments** the router must honor before chasing spot (see §17). They are non-interruptible on the committed slice and require `reliability_tier_required >= standard`.

Reserved demand is rating fuel: contracted GPU-hours move the ABS coupon down (`financial-and-roadmap.md`, covenant target "no platform >60% of fleet revenue"). The router's job is to *satisfy* reserved commitments cheaply (on the right nodes at the right time) while maximizing yield on everything else.

## 15. The interruptible batch / inference backlog (the filler)

The backlog (`source = "backlog"`, `tenant_class: "internal_backlog"` per `data-contracts.md` §4) is Superheat-owned or partner-supplied work — batch inference, fine-tuning queues, embedding/eval runs — that is **preemptible and has no deadline**. It is the utilization filler defined in Part C / §25 (layer L1).

Properties that make it the floor source:
- `interruptible: true`, `preempt_penalty: 0`, `checkpointable: true`, `min_thermal_window_min` very small (5 min) → it fits *any* window.
- Its `net_price_usd_per_gpu_hr` is the **reservation price**, pinned at the **electricity floor = $0.05/GPU-hr** (the marginal cost of running the GPU; calibrated so it sits *below* essentially all external bids). The same $0.05 value is used in the worked example (§16.5) and tied to the live marginal cost from Part C. This guarantees the backlog only wins a window when nothing external does — i.e. it never displaces real revenue, it only prevents idle.
- It is always "available": the router can fall back to it for any node, any window, so windows are never wasted.

Coordination with the utilization engine: the backlog is sized so the queue depth always exceeds fleet idle capacity. If the backlog ever empties, the utilization engine enrolls the node's flexible load in demand-response/VPP (EnergyHub/Renew Home — `financial-and-roadmap.md`) as the absolute floor earner (L4, §25).

## 16. The yield router

The core algorithm. For each node-hour, pick the source with the **highest expected net $/GPU-hr**, subject to thermal window, interruptibility, reliability tier, reserved commitments, and the concentration guard.

### 16.1 Inputs (per candidate source × node)

| Input | Symbol | Source |
|---|---|---|
| Expected gross price | `gross` | connector `quote()` / price feed |
| Source fee | `fee` | `SettlementProfile.fee_pct` |
| FX/token haircut | `fx` | `SettlementProfile.fx_volatility → haircut` |
| Fill probability (will a buyer actually take it this window?) | `p_fill` | connector model (historical fill rate at this ask) |
| Preemption risk (chance the *source* yanks it before window ends) | `p_preempt` | connector model |
| Payout latency penalty | `λ(days)` | discount for time-value + clawback risk |
| Node thermal window (minutes remaining) | `W` | thermal coordinator (§6) / Part C |
| Node reliability tier & network score | `tier, net` | reliability scoring (§4) |
| Current fleet revenue mix | `mix[source]` | metering ledger (§7), rolling 30-day |

### 16.2 Expected value of a source for a node-hour

```
net_price(s)   = gross(s) · (1 − fee(s)) · (1 − fx(s))
latency_factor = 1 / (1 + r_daily · payout_latency_days(s))      # time-value + clawback
E[$/gpu-hr](s, node) =
      net_price(s)
    · p_fill(s, node, ask)               # discovered work actually arrives
    · expected_served_fraction(s, W)     # how much of the window we expect to bill
    · latency_factor(s)
    − expected_preempt_cost(s, node, W)  # checkpoint/restage cost if source preempts
```

**Units / dimensional meaning.** `E[$/gpu-hr]` is the **expected net dollars earned over the candidate window, normalized to a per-GPU-hour-equivalent basis** — i.e. divided by the GPU-hours the window represents. Normalizing to per-GPU-hour is what lets the router compare sources whose candidate windows differ in length (a 45-min vast slot vs a 6-hour reserved block) on the **same** basis; without it, longer windows would spuriously dominate. `expected_served_fraction(s, W) ∈ [0,1]` is the dimensionless fraction of the window we expect to actually bill: `1 − p_preempt(s)·avg_lost_fraction`, clamped so a job needing more than `W` minutes of runnable time can't claim the full window. `expected_preempt_cost` (also $/gpu-hr) uses `connectivity-and-data.md` checkpoint/restage estimates (bandwidth-aware: a node on a slow uplink has higher restage cost, which correctly biases data-heavy jobs toward better-connected nodes).

### 16.3 The decision algorithm (pseudocode)

```python
def route_node_hour(node, window, sources, ledger, policy) -> Assignment:
    # 0. Honor reserved-floor commitments FIRST (see §17).
    #    Reserved jobs are non-interruptible on the committed slice; placement uses the
    #    canonical fits() predicate (data-contracts.md §7), which enforces the whole-job
    #    window-fit for non-interruptible work + tier + confidentiality + gpu_count.
    rsv = pending_reserved_for(node, window, ledger)
    if rsv and fits(rsv, node, window.remaining_s):     # data-contracts.md §7
        return Assignment(job=rsv, reason="reserved_floor")

    # 1. Gather candidate quotes (cheap, cached, parallel across connectors).
    candidates = []
    for s in sources:
        q = s.quote(node, window)                       # SourceQuote
        if not q.available:
            continue
        job_tmpl = s.template_job(node, window)
        # 2. HARD FEASIBILITY FILTER — the canonical fits() predicate (drop, don't score).
        #    fits() covers tier_order, confidentiality_required (trusted_host), gpu_count,
        #    and the interruptible vs non-interruptible window-fit (incl. T_GRACE_S).
        if not fits(job_tmpl, node, window.remaining_s):          continue   # data-contracts.md §7
        ev = expected_value(q, node, window)            # §16.2
        candidates.append((ev, s, q, job_tmpl))

    # 3. CONCENTRATION GUARD — soft steering penalty + HARD 60% covenant stop.
    candidates = apply_concentration_guard(candidates, ledger, policy)

    # 4. Always-available fallback: the interruptible backlog (§15).
    backlog_ev = expected_value(backlog.quote(node, window), node, window)
    candidates.append((backlog_ev, backlog, ..., backlog.template_job(node, window)))

    # 5. Pick the max expected net $/GPU-hr.
    candidates.sort(key=lambda c: c[0], reverse=True)
    ev, src, q, job_tmpl = candidates[0]

    # 6. Commit: list/accept on the chosen source, emit JobAssignment to scheduler.
    job = src.commit(node, window, q)                   # may fail → retry next-best
    return Assignment(job=job, reason=f"max_yield:{src.source_id}", ev=ev)


def apply_concentration_guard(candidates, ledger, policy):
    """Two-layer guard for the 60% single-marketplace limit:
      (a) SOFT steering penalty that grows as a source approaches the cap — steers
          the marginal node-hour toward diversification while approaching 0.60.
      (b) HARD covenant stop at/over policy.max_share (0.60): NO new placement to a
          source already at/over 60% of trailing-30d revenue — UNLESS the only
          alternative is idling the window, in which case we place and FLAG the
          exception for the covenant monitor.
    0.60 is a HARD ABS cash-trap covenant, not a routing preference; the soft
    penalty alone cannot guarantee it (a dominant source can still win on EV)."""
    # Is there any feasible non-over-cap alternative (incl. backlog, which is exempt)?
    has_alt = any(
        ledger.revenue_share(s.source_id, window="30d") < policy.max_share
        or s.tenant_class in EXEMPT_CLASSES                # direct|reserved|internal_backlog
        for ev, s, q, jt in candidates
    )
    out = []
    for ev, s, q, jt in candidates:
        share = ledger.revenue_share(s.source_id, window="30d")
        if s.tenant_class == "marketplace":
            if share >= policy.max_share:                  # 0.60: HARD covenant stop
                if has_alt:
                    continue                               # drop entirely — covenant wins
                ev *= policy.over_cap_factor               # forced placement to avoid idle...
                jt = flag_covenant_exception(jt, s, share) # ...logged/flagged for the monitor
            elif share >= policy.warn_share:               # 0.50: graduated SOFT penalty
                t = (share - policy.warn_share) / (policy.max_share - policy.warn_share)
                ev *= (1 - policy.warn_penalty * t)        # smoothly down to (1 - warn_penalty)
        out.append((ev, s, q, jt))
    return out
```

Notes:
- The guard has **two layers**: a soft steering penalty (the mechanism that kicks in *approaching* the cap) and a **hard stop at the 60% covenant**. The soft penalty alone is insufficient — a marketplace with overwhelming EV could still breach 60% — so the hard stop blocks any new placement to a source already at/over 60% of trailing-30d revenue. The **only** exception is when blocking would idle the node (no other feasible source, not even backlog): we place, but **log and flag the exception** so the covenant monitor sees it. **60% is a hard ABS cash-trap covenant** (`financial-and-roadmap.md`), not just a routing preference; the soft penalty is only the steering mechanism that keeps us from reaching it.
- `direct`, `reserved`, `backlog` are **exempt** from the guard (they reduce, not increase, marketplace dependence) and always provide a non-over-cap alternative (backlog is always commit-able), so the forced-placement exception fires only in pathological cases.
- Step 6 can fail (race: another host took the listing). The router holds the ranked list and retries next-best; if all external fail, the backlog (always commit-able) wins.

### 16.4 Reserved-floor commitments vs spot

(See §17 for the full handling — summarized here as it appears in the algorithm.) Reserved contracts are commitments, not real-time bids, so step 0 of `route_node_hour` places a pending reserved job unconditionally before the spot auction runs.

### 16.5 Worked numeric example (real 4090 rates, Jun 2026)

One H1 node, reliability tier `standard`, good uplink, with a **45-minute** thermal window opening now. Daily discount `r_daily = 0.0005`. Backlog reservation price = $0.05/GPU-hr (the electricity floor — see §15). All `E[$/gpu-hr]` values are net dollars over this window normalized per-GPU-hour-equivalent (§16.2).

| Source | gross $/hr | fee | net $/hr | p_fill | served_frac (45m) | latency days → factor | preempt cost | **E[$/gpu-hr]** |
|---|---|---|---|---|---|---|---|---|
| vast.ai (on-demand) | 0.40 | 0.27 | 0.292 | 0.85 | 0.93 | 3 → 0.9985 | 0.004 | **0.226** |
| RunPod (community) | 0.34 | 0.30 | 0.238 | 0.80 | 0.90 | 4 → 0.998 | 0.004 | **0.167** |
| Salad | 0.18 | ~0.10 eff. | 0.162 | 0.95 | 0.95 | 5 → 0.9975 | 0.003 | **0.143** |
| direct (no fee) | 0.38 | 0.00 | 0.380 | 0.45 | 0.93 | 0 → 1.000 | 0.004 | **0.155** |
| backlog (filler) | 0.05 | 0.00 | 0.050 | 1.00 | 1.00 | 0 → 1.000 | 0.000 | **0.050** |

Calc for vast.ai: `0.292 · 0.85 · 0.93 · 0.9985 = 0.2305`, minus `0.004` preempt cost `≈ 0.226`.

**Decision (no concentration pressure):** vast.ai wins at **$0.226/GPU-hr expected**. Note `direct` has the highest *net* price ($0.38) but low `p_fill` (0.45) right now (thin direct pipeline early on), so its expected value is lower — exactly the trade-off the router exists to make. As the direct pipeline thickens (`p_fill` rises), it overtakes vast.ai at the *same* node-hour with no code change.

**Now apply the concentration guard.** Suppose vast.ai is already at **62%** of trailing-30d fleet revenue (above the 0.60 cap). Because 62% is at/over the **hard 60% covenant** (§16.3), the portfolio-level hard stop blocks any *new* vast.ai placement here outright (there is a non-idling alternative), so vast.ai is dropped — the soft `over_cap_factor = 0.5` would only have mattered while *approaching* the cap. Either way:

- vast.ai dropped by the hard stop (and would be `0.226 · 0.5 = 0.113` under the soft penalty alone — still below RunPod).
- RunPod unchanged: `0.167` → **now wins**.

The router transparently shifts this node-hour to RunPod, pulling vast.ai's share back down toward the 60% line — while still earning $0.167 rather than idling. Over many node-hours this is what holds the fleet under the securitization covenant.

**Reserved-floor interaction.** If this node had a pending reserved commitment (net $0.49, tier `standard`, fits 45m), step 0 would have placed it *before* the auction ran — $0.49 dominates everything above anyway, so honoring the contract is also the yield-optimal choice this hour.

## 17. Reserved-floor commitments vs spot (detail)

Reserved contracts are commitments, not real-time bids, so they are handled *before* the spot auction:

1. **Forecast & reserve.** A nightly planner (in Part A) looks at each reserved contract's monthly GPU-hour commitment and the fleet's forecasted thermal windows, and **pre-allocates** enough high-reliability node-hours to meet it — spread across the month, not front-loaded — leaving the rest free for spot.
2. **Just-in-time honor.** When a pre-allocated window opens, `route_node_hour` step 0 places the reserved job unconditionally (it has already been "paid for" via the contract).
3. **Opportunistic over-delivery.** If reserved demand is light in a given hour, that pre-allocation is *released back* to the spot auction — reserved is a floor, never a ceiling on utilization.
4. **Pricing comparison.** Reserved net (~$0.49, no fee) usually beats spot net (~$0.29 after marketplace fee), so honoring reserved is also yield-optimal most hours — the explicit step-0 handling only matters in the rare hour when a fleeting spot spike would otherwise tempt the router to skip a commitment.

## 18. Settlement reconciliation

Marketplace payouts are noisy, lagged, fee-laden, and sometimes in tokens. The metering ledger (§7) is the **authoritative, per-unit record that becomes the securitization tape** — so every external dollar must be tied back to the GPU that earned it. Reconciliation is **signed usage records vs marketplace settlement (money), per GPU** — NOT lossy telemetry, which is for health/scoring only (`data-contracts.md` §8; §7.4).

### 18.1 The two records that must reconcile

1. **Signed usage record** (we generate — the canonical `superheat.usage.v1`, `data-contracts.md` §8; ledger table §7.2): **per-GPU** (`gpu_slot` mandatory), keyed by `job_id`, with `start_ts`/`end_ts` (never spanning a billing-period boundary), `gpu_seconds`, `energy_kwh`, and `gross_usd`/`fee_usd`/`net_usd`. It is **double-signed**: the node identity key (`node_sig`) signs the metered facts at delivery; the control plane counter-signs (`cloud_sig`) after this reconciliation step — that counter-signature is what makes the tape provenance chain real.
2. **Marketplace settlement** (the source generates, lagged): `connector.pull_settlement()` returns `PayoutRecord`s keyed by `source_job_ref`, with the *realized* amount, realized fee, currency, FX rate, and dispute status. This is the money side.

### 18.2 The reconciliation pipeline

The ±tolerance applies to **signed-usage-record `net_usd` vs marketplace settlement** (`data-contracts.md` §8), aggregated per source. Per-GPU usage records are summed (joining on `node_id` AND `gpu_slot`) before matching to the marketplace statement.

```
pull_settlement()  ──►  match PayoutRecord.source_job_ref  ──►  usage_record.job_id / source_job_ref
                            │
                            ├─ matched & Σ net_usd agree (±tolerance) → mark RECONCILED,
                            │     control plane writes cloud_sig (double-signs the usage record),
                            │     post realized net to node's owner-settlement sub-ledger
                            │
                            ├─ matched, amount differs > tolerance → FLAG variance
                            │     (fee drift, partial-hour rounding, mid-job price change)
                            │
                            ├─ payout with no usage record → ORPHAN (investigate:
                            │     clock skew, missed record, duplicate payout)
                            │
                            └─ usage record with no payout past SLA window → MISSING
                                  (source dispute, clawback, non-payment) → arrears report
```

Key handling:
- **Currency normalization.** Token payouts (io.net, Akash) are converted to USD at the realized settlement-time FX and stored both ways. The `fx_haircut` the router applied at decision time is reconciled against actual slippage; persistent under/over-estimation feeds back into the connector's haircut model.
- **Fee truth-up.** The realized `fee_pct` from the payout updates the connector's `SettlementProfile` (marketplaces change commissions). This closes the loop so the router's future net-price estimates stay accurate.
- **Owner settlement & servicing fee.** Reconciled net revenue is split per the financing/tranche tag on the node (registry, §1): owner payout, Superheat servicing fee (0.5–2%/yr of node value — `financial-and-roadmap.md`), and platform take. This per-unit split is what makes the tape auditable.
- **Dispute/clawback window.** Revenue is `RECONCILED` but not `FINAL` until `dispute_window_days` elapses; the tape reports both seasoned-final and provisional figures so underwriters can haircut appropriately.

### 18.3 What goes on the securitization tape

For each node, per period: GPU-hours delivered, weighted-avg realized net $/GPU-hr, revenue by source (with the concentration share that the guard enforced), reconciliation status mix (% reconciled-final vs provisional vs arrears), and reliability/uptime. This is the clean, per-unit, source-diversified ledger that `financial-and-roadmap.md` requires for warehouse and ABS underwriting. (The materialized-view mechanics that produce per-node GPU-hours and utilization are §7.3.)

## 19. Demand-layer build phasing & open questions

**Build phasing** (aligns with `financial-and-roadmap.md`):

- **Phase A — vast.ai connector only.** Implement `MarketplaceConnector` for vast.ai, the canonical `superheat.job.v1` schema (`data-contracts.md` §4), and a trivial single-source router (always vast). Settlement reconciliation v0 (USD, signed usage records vs marketplace statement, manual variance review). Produces the first metered tape.
- **Phase B — multi-source router + backlog.** Add RunPod + Salad connectors, the full yield router (§16.3) with the concentration guard, the interruptible backlog filler, and automated reconciliation. This is where margin capture and the utilization floor turn on.
- **Phase C — own demand + DePIN.** Direct/enterprise API (Part D), reserved contracts and the reserved-floor planner (§17), io.net + Akash (with token FX handling in §18), and the full per-unit securitization tape feed.

**Open questions** (for `financial-and-roadmap.md`):

- **Price-feed freshness vs API cost.** Marketplaces rate-limit price queries; `quote()` must lean on cached feeds + a learned price model, refreshed opportunistically. How stale is too stale before the router mis-ranks?
- **Listing-race loss rate.** Step-6 commit failures (another host grabbed the rent) waste a routing cycle. Measure the real loss rate per source; pre-list on the top-2 sources and cancel the loser if churn is high.
- **Backlog reservation price calibration.** Set too high, the backlog steals windows from thin-but-real external demand; too low, it never protects the electricity floor. Tie it to the live marginal cost from Part C.
- **Concentration penalty shape.** Linear graduated penalty is a starting point; if it over-corrects (whipsawing share), move to a controller that targets the share directly.
- **Token-payout volatility.** io.net/Akash FX haircut and claim latency may make their *expected* net uncompetitive after risk-adjustment — validate that they ever win a node-hour before investing further in those connectors.

---

# Part C — Utilization engine

The joint compute-and-heat optimization that keeps every heater-embedded GPU as close to fully booked as the thermal and comfort constraints physically allow. This Part turns the economics memo's headline assumption — **60% GPU utilization × 70% thermal-demand overlap** (`financial-and-roadmap.md`; `../market-research/Superheat_Recalculated_Economics_and_Debt_Business_Plan.md` §2.3 base case ★) — into a control system, and tests whether that assumption is honest.

## 20. The tank-as-thermal-battery model

### 20.0 The problem in one paragraph

A Superheat node can only run compute when its heat is **wanted now** or can be **stored for later**. Idle GPU-hours are lost twice: lost rental revenue *and* lost heat the customer must buy from the grid anyway. The tank is a thermal battery, so "wanted now" is not the binding constraint — "wanted now **or** storable" is. The utilization engine's whole job is to keep the GPU busy across that wider window while never, ever compromising the comfort floor (Principle 2: *fail safe to a water heater*). Everything below is the math and control logic for that. (The control-plane interface that exposes this model to the scheduler is §6.)

### 20.1 State variables (per node, per control interval)

The thermal coordinator (§6) maintains a per-node thermal state. The control interval is **5 minutes** (`Δt = 1/12 h`); the planning horizon is **6 hours rolling** (`H = 72` intervals).

| Symbol | Meaning | Units | Source |
|---|---|---|---|
| `T_tank` | Current bulk tank temperature | °C | node sensor (`node.md`) |
| `T_set` | Comfort setpoint (customer-chosen) | °C | config; typ. 50–60 °C |
| `T_min` | Comfort floor — never deliver below this | °C | safety; typ. `T_set − 5` |
| `T_max` | Tank/material safety ceiling | °C | hardware; typ. 70–82 °C |
| `T_dr` | DR/storage charge target (above `T_set`, below `T_max`) | °C | policy; e.g. 70 °C |
| `C_th` | Tank thermal capacitance | kWh/°C | `m·c_p`; ≈ 0.352 kWh/°C for 80 gal (data-contracts §6) |
| `E_store` | **Usable headroom** = `C_th·(T_max − T_tank)` | kWh | derived; full-band ≈ 8.8 kWh (data-contracts §6) |
| `E_avail` | **Stored usable heat** = `C_th·(T_tank − T_min)` | kWh | derived |
| `D(t)` | Heat-demand forecast (draw) over horizon | kW | forecaster, §20.4 |
| `loss(t)` | Standby loss to ambient | kW | `UA·(T_tank − T_amb)` |
| `P_gpu` | GPU electrical/thermal power when running | kW | 0.412–0.450 kW/GPU |
| `P_elem` | Resistive backup element power | kW | up to ~3.3 kW system |

> **Worked capacitance.** An 80-gallon tank ≈ 303 kg water; `c_p = 4.186 kJ/kg·°C = 1.163e-3 kWh/kg·°C`. So `C_th ≈ 303 × 1.163e-3 ≈ 0.352 kWh/°C` — the canonical value (data-contracts §6); a 50-gallon tank ≈ `0.22 kWh/°C`. Over a usable band of `T_max − T_min = 70 − 45 = 25 °C`, an 80-gal tank banks **≈ 8.8 kWh** (canonical `E_store`, data-contracts §6) — roughly **21 GPU-hours** of a single 412 W card, or ~2.7 hours of an 8× node. That stored band is the decoupling buffer.

### 20.2 Charge / discharge dynamics

The tank temperature evolves each interval as a first-order energy balance:

```
T_tank(t+Δt) = T_tank(t)
             + ( P_in(t) − D(t) − loss(t) ) · Δt / C_th

  where  P_in(t) = η_capture · P_gpu · n_running        (compute-as-heat)
                 + P_elem(t)                            (resistive backup)
```

`η_capture` is the fraction of GPU electrical draw delivered into the tank water (heat-exchanger/coldplate efficiency; assume ~0.95 — near-1.0 PUE is the entire thesis, `system-design-overview.md` §3.3). **`η_capture ≈ 0.95` is an unvalidated placeholder (data-contracts §6); every duty-cycle conclusion in this doc is conditional on measuring it in Phase A.** `D(t)` discharges the tank; `loss(t)` is standby bleed (also a discharge, but small).

Two regimes drive scheduling:
- **Discharge-led (heat wanted now):** `D(t) > P_in` → running the GPU directly offsets a draw the customer would otherwise pay grid rates for. This is the *free-electricity* hour. Always run if work exists.
- **Charge-led (heat storable):** `D(t) ≈ 0` but `T_tank < T_max` → run the GPU to **bank** heat into `E_store`, decoupling compute from instantaneous demand. Runnable until headroom is exhausted.

### 20.3 Pre-heat scheduling

Because storage is finite (~8.8 kWh above), naive "charge whenever idle" fills the tank early and then strands the GPU. The smarter move is **pre-heat ahead of forecasted demand**: deliberately keep headroom in reserve so a *known* upcoming draw can be served by compute heat rather than by the resistive element.

Pre-heat solves: *given forecast `D(t)` over the horizon, what is the latest interval to start charging so the tank reaches the level needed to cover the next demand peak entirely with GPU heat?* This converts a future draw (which would otherwise be element-served, or worse, a moment of no compute) into bookable GPU-hours, and it is what lets the engine fill the **morning/evening DHW peaks** with compute instead of resistance.

```
preheat_start(peak_t, peak_kWh):
    # FEASIBILITY GUARD: the tank can bank at most E_store of headroom.
    # Pre-heat can only cover the part of the peak the tank can physically hold;
    # the remainder must come from real-time GPU heat during the peak or the element.
    # E_store here is current usable headroom = C_th·(T_max − T_tank).
    bankable_kWh = min(peak_kWh, E_store)             # can't store more than the tank holds
    # account for standby loss bled off over the pre-heat interval (heat banked early decays):
    needed = bankable_kWh / (η_capture · P_gpu · n_gpu)        # GPU-hours to charge it
    loss_makeup = avg_loss_kW · needed / (η_capture · P_gpu · n_gpu)  # extra GPU-hrs to offset bleed
    needed += loss_makeup
    if bankable_kWh < peak_kWh:
        # peak exceeds storage; pre-heat fills the tank, peak-time compute + element cover the rest
        flag PARTIAL_PREHEAT(shortfall = peak_kWh − bankable_kWh)
    return peak_t − needed − safety_margin
```

The guard ensures pre-heat never schedules banking more heat than `E_store` holds (an unguarded formula returns a start time for an impossible charge when `peak_kWh > E_store`). When the peak exceeds storage, pre-heat fills the tank and the residual is served by GPU heat *during* the peak (discharge-led) or by the resistive element — it is not "banked early."

The thermal coordinator emits, per node per interval, a **runnable-minutes budget** and a **mode hint** (`discharge` / `charge` / `preheat` / `hold`). The scheduler in Part A (§5–§6) consumes that; it never computes thermal feasibility itself. Hard comfort/safety limits (`T_min`, `T_max`) are enforced **locally on the node** by the agent and always override the cloud plan (`system-design-overview.md` §4.2).

### 20.4 The heat-demand forecast

`D(t)` is per-site and learned: a baseline hot-water draw profile (occupancy- and weekday-keyed) plus, in cold climates, a space-heating load driven by the weather forecast (HDD). Commercial sites (hotels, apartments, campuses) have **near-continuous, high-confidence** draw — which is exactly why the economics memo calls commercial "the financeable asset class" (§2.1) and why its 70% overlap is a commercial number. Residential draw is spikier and lower-volume; see §28.

## 21. The thermal opportunity window

### 21.1 Computing runnable minutes for the horizon

For each interval the GPU is runnable if running it keeps the tank inside `[T_min, T_max]` over the horizon given forecast draw. The window is the count of feasible intervals:

```
thermal_window(node, horizon):
    runnable = 0
    T = T_tank
    for t in horizon:                      # 5-min steps
        headroom = C_th · (T_max − T)
        # heat we'd add this interval if GPU runs:
        q_add = η_capture · P_gpu · n_gpu · Δt
        if q_add <= headroom + D(t)·Δt:    # draw makes room too
            runnable += 1
            T += (η_capture·P_gpu·n_gpu − D(t) − loss(t)) · Δt / C_th
        else:
            # tank would overheat — must idle or shed to element-off
            T += (− D(t) − loss(t)) · Δt / C_th
        T = clamp(T, T_min, T_max)
    return runnable · Δt                    # runnable HOURS in the horizon
```

The window is the sum of (a) **discharge** intervals where `D(t)` carries the heat away, plus (b) **charge** intervals while headroom lasts. Storage (`E_store`) is what makes (b) nonzero when `D(t) = 0` — that is the battery doing its job.

### 21.2 Duty-cycle math: 412–450 W GPU against a 3.3 kW system

The single 4090 home unit (H1) is the clearest case. The thermal *system* is rated 3.3 kW, but one 4090 only delivers **0.412–0.450 kW** of that as compute heat; the rest of the system's heating capacity is the resistive element, which exists to guarantee comfort (Principle 2) and is **not** a compute path.

So the relevant ratio is not "GPU vs 3.3 kW." It's **how many of the site's heat-demand kWh can the small GPU heat source actually supply.** Define the site's daily thermal load `E_day` (kWh/day) and the GPU's max daily heat delivery:

```
GPU_max_daily = η_capture · P_gpu · 24 h
             = 0.95 × 0.412 kW × 24 ≈ 9.4 kWh/day (single 4090)
```

- **Discharge-coverable fraction** = `min(1, GPU_max_daily / E_day)`. If the site needs more heat per day than one GPU can make, the GPU can run *continuously* and still all of its heat is consumed → the thermal constraint is **not** binding; utilization is then capped only by *demand for compute*, not by heat. This is the cold-climate / commercial regime.
- If `E_day < GPU_max_daily`, the GPU would overproduce heat; the excess must be **banked then bled** (`E_store` + standby loss) or the GPU must **duty-cycle down**. The runnable fraction becomes:

```
runnable_fraction ≈ (E_day + E_store_cycled_per_day) / GPU_max_daily,   capped at 1.0
```

where `E_store_cycled_per_day` is how many times per day the tank can charge-and-discharge its usable band (bounded by draw events resetting headroom).

## 22. Reconciling to 60% × 70%

The economics base case multiplies two independent factors (`Superheat_Recalculated_Economics_and_Debt_Business_Plan.md` §2.3):

1. **70% thermal overlap** — the fraction of GPU-hours whose heat lands during (or is bankable into) a real heat-demand window, i.e. the *free-electricity* fraction. The other 30% of run-hours pay retail electricity (~$0.07/GPU-hr at $0.17/kWh, §2.1) but are still margin-positive when rented.
2. **60% utilization** — the fraction of clock-hours the GPU is actually rented and running, gated by **compute demand**, not heat.

These compose as `0.60 × 0.70 = 0.42` of all clock-hours are *both rented and free-heat*; the remaining `0.60 × 0.30 = 0.18` are rented-but-paid-electricity, and `0.40` are unrented. The utilization engine attacks **both** the 40% unrented gap (via filler + DR, §25) **and** the 30% paid-electricity slice (via pre-heat, shifting run-hours into demand windows, §20.3).

**Sanity check against the thermal ceiling.** A commercial 8× node delivers `0.95 × 8 × 0.412 × 24 ≈ 75 kWh/day` of heat. A hotel/apartment site with near-continuous DHW easily absorbs that (the memo's §2.1 notes a *single home* absorbs only ~16% of a unit's max thermal output — which is precisely why residential DHW-only nets just ~$2.3K/yr and commercial nets ~$11.7K). So for the **commercial base case the thermal constraint is loose**: 60% utilization is demand-limited, and 70% of those hours overlapping real heat demand is achievable because commercial draw is nearly always on. The 60%×70% figures are **internally consistent for commercial**, and the engine's filler/DR layers exist to push the 60% upward over time (`financial-and-roadmap.md`).

**Fleet-weighted reconciliation (the honest version).** 60%×70% is a *commercial* number; the blended fleet does not clear it unless commercial dominates the mix. Modeling each segment with a realistic thermal-overlap and an achievable utilization (annual, seasonal-weighted per §28.3), then weighting by an illustrative fleet mix:

| Fleet segment | Thermal overlap | Achievable utilization | overlap × util | Notes |
|---|---|---|---|---|
| Commercial (continuous DHW / hydronic) | ~70% | ~60% | **0.42** | the financeable base case; thermal constraint loose year-round (§21.2, §28.1). Headline **≈ $11.7K/unit-yr**. |
| Residential cold-climate (DHW + space heat) | ~60% | ~55% | **0.33** | space heat keeps `E_day > GPU_max_daily` most of the year; ≈ **$8.5K/unit-yr with space heat** (overview). |
| Residential warm-climate / DHW-only | ~40% | ~45% | **0.18** | GPU overproduces vs a DHW-only load (§21.2, §28.2); tank saturates, run-hours go paid-electricity or unavailable. ≈ **$2.3K/unit-yr DHW-only** (overview). |

- **All-commercial fleet:** `0.42` — meets the 60%×70% base case exactly. ✓
- **Even split (⅓ each):** `(0.42 + 0.33 + 0.18)/3 ≈ 0.31` — **below** the 0.42 base case. The two residential segments drag the blend down.
- **Commercial-heavy (60% comm / 25% cold-res / 15% warm-res):** `0.60·0.42 + 0.25·0.33 + 0.15·0.18 ≈ 0.36` — still short of 0.42.

**Conclusion:** **60%×70% (= 0.42) is a COMMERCIAL figure, not a blended-fleet figure.** Any fleet with material residential exposure clears it only if commercial weight is high; a balanced fleet lands closer to **0.31–0.36**. The headline **≈ $11.7K/unit-yr is likewise a commercial number** — residential is **≈ $2.3K/unit-yr DHW-only** or **≈ $8.5K/unit-yr with space heat** (`system-design-overview.md`; economics memo). Utilization (and revenue) promised to financiers must be the **fleet-mix-weighted, regional, seasonal-weighted** number per §28.3 — never the commercial single-segment figure. Residential is *optimistic* against 60%×70% — see §28.

## 23. (folded into §22)

*The 60%×70% reconciliation above is the engine's headline number; the per-region/seasonal modeling that feeds the financing tape is §28.3.*

## 24. Reliability tiering (utilization-engine view)

Heterogeneous, residential-NAT'd nodes cannot all promise the same uptime. We **measure, then price** reliability (Principle 4) and route work by what each node can honestly promise. The **canonical tier enum (`premium/standard/spot/probation`) and the reliability score that derives it are defined in data-contracts §2/§3** and computed by the reliability scoring service (§4) — this section does not redefine them; it only maps the uptime thresholds the engine watches onto those names. Tiering is consumed by both the scheduler (§5) and the yield router (§16).

### 24.1 Tier definitions and thresholds

The canonical tiers (data-contracts §2) are derived from the reliability **score** (data-contracts §3, §4.2), not from uptime alone; uptime is the dominant input. The trailing-30d-uptime bands below are the engine's working thresholds, mapped to the canonical names:

| Tier (canonical, data-contracts §2) | Trailing-30d uptime | Network (upload, jitter) | Thermal-window stability | Eligible work |
|---|---|---|---|---|
| **`premium`** | ≥ 99.0% | ≥ 50 Mbps up, low jitter, rare relay | predictable long windows (commercial, continuous draw) | L3 reserved/premium/non-interruptible **+** all lower |
| **`standard`** | 95.0–99.0% | ≥ 20 Mbps up | moderate windows | L2 spot + L1 filler |
| **`spot`** | 90.0–95.0% | variable / CGNAT-relayed | short or spiky windows (residential DHW) | L1 filler + short L2 interruptible only |
| **`probation`** | < 90% or failing thermal SLA, **or** unseasoned (< 30 d history) | — | — | filler only; no paid SLA; candidate for service ticket |

Thresholds are **starting points**, tuned as tape accumulates; the authoritative score bands and the seasoning gate live in data-contracts §3 (and §4.2). The tiering directly produces the *seasoned, high-reliability sub-pools* the securitization tape wants (`financial-and-roadmap.md`; `Superheat_Recalculated_Economics_and_Debt_Business_Plan.md` §3.5 covenants). A node's tier is an EWMA-smoothed input so a single blip doesn't whipsaw routing.

### 24.2 Routing rule

Routing reuses the canonical feasibility predicate `fits(job, node, window_remaining_s)` (data-contracts §7) rather than a local check — it already enforces the tier-order comparison (data-contracts §2), the confidentiality match, and the non-interruptible window-fit rule:

```
route(job, node):
    if not fits(job, node, thermal_window(node, job.duration)):  return REJECT
    return ACCEPT
```

Premium/contracted/non-interruptible jobs land on `premium` nodes with a confirmed window that covers the whole job. Everything spot/interruptible plus the filler flows to `standard`/`spot` nodes, where preemption is expected and made safe by checkpoint/resume (§26). This maximizes *total fleet revenue* without over-promising on flaky nodes.

## 25. The four-layer utilization stack

Each node-hour is filled by the first layer that pays, top-down. The yield router (§16) arbitrates layers 1–3 by `$/GPU-hr`; the thermal coordinator (§6) gates all of them; layer 4 catches the rest.

```
                       value / $ per GPU-hr
   ┌────────────────────────────────────────────────────────┐  high
   │  L3  RESERVED-INSTANCE FLOOR   contracted, non-interrupt │
   │      premium / enterprise direct                         │
   ├────────────────────────────────────────────────────────┤
   │  L2  SPOT / MARKETPLACE        on-demand & interruptible │
   │      (vast.ai, RunPod, Salad …)  yield-routed            │
   ├────────────────────────────────────────────────────────┤
   │  L1  INTERRUPTIBLE OWN-BACKLOG FILLER                    │
   │      always-something-to-run; preemptible; no deadline   │
   ├────────────────────────────────────────────────────────┤
   │  L0  THERMAL BATTERY + PRE-HEAT  (enabling substrate)    │  the
   │      decouples compute from instantaneous heat demand    │  unlock
   └────────────────────────────────────────────────────────┘
        when NEITHER compute NOR heat is wanted ↓
   ┌────────────────────────────────────────────────────────┐
   │  L4  DEMAND-RESPONSE / VPP      earn on the flexible load│
   │      EnergyHub · Renew Home · Voltus · Leap             │
   └────────────────────────────────────────────────────────┘
```

**L0 — Thermal battery + pre-heat (the substrate, §20).** Not a revenue layer; it *widens the window* every other layer runs inside. Without it, compute waits on instantaneous demand and utilization collapses to the DHW duty cycle.

**L1 — Interruptible own-backlog filler (§15).** Superheat-owned or partner batch/inference work that is preemptible and deadline-free. Its purpose is a **floor guarantee**: whenever a node has a thermal window and no better-paying job, there is *always something to run*, so no window is wasted. The filler is intentionally low-value — it exists to convert otherwise-idle thermal windows into nonzero revenue and into useful heat the customer would otherwise buy. It is the layer that turns "60% utilization" into a number the engine *defends* rather than merely *hopes for*.

**L2 — Spot / marketplace.** Yield-routed on-demand and interruptible jobs from vast.ai, RunPod, Salad, etc. (§12). Higher value than filler, fully preemptible (or short-lived). The router prefers these over L1 whenever available.

**L3 — Reserved-instance floor.** Contracted/reserved demand and direct enterprise jobs (§14). Highest value, **non-interruptible** — therefore only placed on high-reliability nodes with a *confirmed* thermal window long enough to finish (or safely checkpoint). Provides revenue stability and the contracted-revenue ratio the financing engine rewards (`financial-and-roadmap.md`).

**L4 — Demand-response / VPP.** For hours when **neither compute nor heat is wanted** (tank full, no draw forecast, no profitable job) — instead of idle-and-coast, enroll the node's *flexible load* in DR/VPP programs and earn $20–100/unit-yr of utility-grade contracted income. Partners: **EnergyHub** (already aggregates Rheem water heaters across 30+ utilities — integration is effectively pre-built for our device class), **Renew Home** (Google-backed), Voltus, CPower, Leap (`../market-research/Superheat_Partnership_and_Capital_Map.md` §6). A DR *charge* event (utility pays us to soak grid energy) is a chance to **pre-heat for free**; a DR *shed* event (utility pays us to drop load) means GPU-off — which is itself the comfort-safe default. DR and compute never fight: DR only claims hours the upper layers declined.

## 26. Preemption & graceful-stop mechanics

Interruptible work *will* be preempted — the heat window ends, a higher-yield L2/L3 job arrives, a DR shed event fires, or local comfort safety trips. Preemption is only safe because of checkpoint/resume (`connectivity-and-data.md`). The engine guarantees a **graceful-stop budget** before any forced stop.

### 26.1 Preemption triggers (priority order)

1. **Comfort/safety (local, instant, non-negotiable):** `T_tank` hits `T_max`, or the resistive element must fire to hold `T_min` and the GPU heat is in the way. The node agent acts *locally* without waiting for the cloud (`system-design-overview.md` §4.2). Comfort always wins.
2. **Thermal-window expiry (planned):** the coordinator's window for this node is closing. Signalled `≥ T_grace` ahead so the job checkpoints, not crashes.
3. **Yield preemption (planned):** a higher-value job (L2 over L1, L3 over L2) wants the node. Router-initiated; honor `T_grace`.
4. **DR shed event (semi-planned):** utility calls a load drop; usually minutes of notice → ample grace.

### 26.2 Graceful-stop protocol

```
graceful_stop(job, reason, deadline):
    signal(job, PREEMPT, reason)               # SIGTERM-style, with deadline
    wait_for_checkpoint(job, T_grace)          # job flushes state to local NVMe
    if checkpointed_before(deadline):
        mark RESUMABLE; register checkpoint w/ connectivity-and-data     # resume elsewhere
    else if reason == COMFORT_SAFETY:
        hard_kill(job)                         # comfort is absolute; lose the work
        mark FAILED_PREEMPT (filler) / REQUEUE (idempotent)
    else:
        extend_or_kill(job)                    # bounded extension for spot/premium
    set_gpu_power_state(OFF / THROTTLE)        # heat path closes
```

- **`T_grace` budget:** filler/spot jobs get a small budget (`T_GRACE_S = 30` s, canonical in data-contracts §7, owned by `connectivity-and-data.md`) to flush an in-flight checkpoint to fast local NVMe; the heater never waits on compute. Because 30 s cannot finish a multi-minute upload, the durable resume path is the last PoP-persisted checkpoint, not the in-flight flush (data-contracts §7). Non-interruptible L3 jobs are *only* scheduled where the window comfortably exceeds runtime, so they should rarely hit a planned preemption at all.
- **Checkpoint cadence** is set so the *expected* lost work per preemption is small relative to throughput — interruptible jobs checkpoint frequently; this is a `connectivity-and-data.md` policy keyed off the node's tier and historical window length.
- **Resume** is location-independent: the checkpoint is content-addressed and may resume on *another* node with a window (`connectivity-and-data.md` regional cache + bandwidth-aware placement). Preemption on a Superheat node is a *migration*, not a loss.

The contract with renters: interruptible/filler work carries **clear eviction semantics** (what's kept, for how long) so preemption is never a surprise (`connectivity-and-data.md`; the renter POV is Part D / §35). Premium L3 work carries an SLA that the tiering (§24) is engineered to honor.

## 27. (folded into §26)

*Preemption triggers and the graceful-stop protocol are §26; the renter-facing interruption UX, lifecycle, and per-tier SLOs are Part D / §34–§35.*

## 28. Seasonality & climate

Utilization is bounded by heat demand, and heat demand swings with season and climate. The 70% overlap is a **commercial annual average**; it is not uniform across the year or the map, and pretending otherwise would mis-underwrite the tape (`financial-and-roadmap.md`; economics memo §3.8 *thermal-demand shortfall*).

### 28.1 The three regimes

| Regime | Thermal load | Engine behavior | Realized overlap |
|---|---|---|---|
| **Cold-climate + space heating (winter)** | very high (hydronic + DHW; ~12,000 kWh/yr homes, continuous commercial) | thermal constraint **loose** → utilization is demand-limited; nearly all run-hours are free-heat | high (→ 80–90%) |
| **Shoulder seasons** | moderate DHW + intermittent space heat | mixed; pre-heat + storage carry most run-hours | medium (~60–70%) |
| **Summer / warm-climate DHW-only** | low (~4,500 kWh/yr residential) | thermal constraint **tight** → tank fills fast; GPU must duty-cycle or rely on DR/coast; many run-hours pay retail electricity | **low (→ 30–50%)** |

### 28.2 The honest risk

In **summer and in warm climates, a small GPU heat source can overproduce** relative to a DHW-only load (§21.2: one 4090 makes ~9.4 kWh/day; a home needs ~12 kWh/day of DHW in winter but far less hot water and *no* space heat in summer). When `E_day` falls below `GPU_max_daily`, the tank saturates, run-hours that aren't rented-and-free become rented-and-paid (eroding the 70%) or simply unavailable (eroding the 60%). This is the structural reason the economics memo underwrites **commercial-first** and rates *Residential DHW-only* at just ~$2.3K/yr net (−93% vs deck).

**Engine mitigations, in priority:**
1. **Commercial-first deployment** — near-continuous draw keeps the constraint loose year-round (the financeable asset class).
2. **Geographic/seasonal load balancing** — the yield router can steer scarce premium jobs toward nodes *currently* in a high-overlap regime (cold regions in winter), while warm/summer nodes lean on filler + DR. The fleet's geographic and seasonal diversity smooths *aggregate* utilization even when individual nodes swing.
3. **DR/VPP as the summer floor (L4)** — when neither heat nor profitable compute is wanted, DR income replaces lost thermal-overlap revenue. This is exactly when VPP value (summer peak shaving) is highest.
4. **Pre-heat overnight, discharge into morning/evening peaks** — maximizes the fraction of DHW load served by compute even in low-demand seasons.

### 28.3 Per-region utilization modeling

The control plane maintains a **per-region, per-month overlap model** (HDD-driven for space heat, occupancy-driven for DHW) feeding both the forecaster (§20.4) and the financing tape. Utilization promised to financiers must be the *regional, seasonal-weighted* number, not the winter peak. `financial-and-roadmap.md` owns this risk register entry; the engine owns producing the measured inputs.

## 29. The control loop (per node, per interval)

This is the decision the engine makes every 5 minutes for every node. Thermal feasibility comes from §20–§21; layer selection from §25; tiering from §24; preemption from §26.

```
decide(node, interval):
    # --- 0. Local safety first (agent-enforced, always wins) ---
    if T_tank >= T_max or comfort_at_risk(node):
        if running(node): graceful_stop(job, COMFORT_SAFETY, now)
        set_gpu_power_state(OFF)
        ensure_comfort(node)                     # element may fire; heater guaranteed
        return IDLE_SAFETY

    # --- 1. Thermal window for the horizon ---
    window = thermal_window(node, HORIZON)       # runnable hours, §21.1
    mode   = coordinator_mode_hint(node)         # discharge|charge|preheat|hold

    # --- 2. Is there heat demand NOW or bankable headroom? ---
    runnable_now = (D(now) > 0) or (E_store(node) > q_add_interval)
    if not runnable_now and mode == 'hold':
        # neither heat now nor headroom to bank it
        if dr_event_active(node):
            return run_DR_event(node)            # L4: shed or charge, earn anyway
        return IDLE_AND_COAST                     # rare; nothing wanted, nothing earns

    # --- 3. Pre-heat reservation: don't fill the tank too early ---
    if mode == 'preheat' and overproducing(node):
        # keep headroom for forecasted peak; only run if a paying job exists
        if not best_job_available(node):
            return HOLD_HEADROOM

    # --- 4. Pick the highest-value runnable layer (yield router) ---
    job = yield_router.best_job(node, window, node.tier)   # §16
        # tries L3 (if premium & window covers runtime) -> L2 -> L1 filler

    if job is None:
        # router found nothing payable, but heat is wanted/storable:
        job = backlog.next_filler(node)          # L1: ALWAYS something to run
    if job is None:
        if dr_event_active(node): return run_DR_event(node)   # L4 fallback
        return IDLE_AND_COAST

    # --- 5. Preempt lower-value work if a better job arrived ---
    if running(node) and job.value > current_job(node).value + HYSTERESIS:
        graceful_stop(current_job(node), YIELD_PREEMPT, now + T_grace)

    # --- 6. Run, capturing heat into the tank ---
    set_gpu_power_state(ON)
    run(job, node)                               # heat -> tank (charge or discharge)
    meter(job, node)                             # -> securitization ledger (§7)
    return RUNNING(job.layer)
```

Notes on the loop:
- **`IDLE_AND_COAST` is the failure state the whole engine minimizes.** Reaching it means: no heat wanted, no headroom, no payable job, no filler, no DR. The L1 filler is specifically designed so the `backlog.next_filler` branch almost never returns `None` (Principle: always something to run). In practice the only legitimate idle is *tank full + no compute demand at all* — which L4 DR is meant to monetize.
- **Hysteresis** on yield preemption (step 5) prevents thrashing between similarly-valued jobs, which would waste checkpoint I/O on slow consumer uplinks. (This is the same anti-thrash discipline the scheduler applies — §5.2.)
- **Metering** (step 6) writes to the per-unit ledger that *is* the securitization tape (§7; economics memo §3.5) — every booked GPU-hour, its layer, its `$/GPU-hr`, and whether its heat was free-overlap or paid.

## 30. Worked duty-cycle example

**Setup.** One H1 home node, single RTX 4090 @ 412 W, `η_capture = 0.95`, 80-gal tank (`C_th = 0.352 kWh/°C`), band `T_min = 45 °C → T_max = 70 °C` ⇒ `E_store_band ≈ 8.8 kWh`. Cold-climate winter day; space-heating + DHW load `E_day = 16 kWh/day`, concentrated in a morning peak (06:00–08:00) and evening peak (18:00–22:00). GPU max heat delivery `= 0.95 × 0.412 × 24 ≈ 9.4 kWh/day`.

**Observation.** `E_day (16) > GPU_max_daily (9.4)` ⇒ the GPU **cannot overproduce**; every kWh it makes is consumed. The thermal constraint is **loose**: the GPU can run **24/7** and 100% of its heat is free-overlap. Utilization is now capped only by *compute demand*, not heat.

**Hour-by-hour (sketch):**

| Window | `D` (heat) | Tank | Engine action | Layer | Heat status |
|---|---|---|---|---|---|
| 00:00–05:00 | low (standby + trickle) | charging toward `T_max` | run if job; pre-heat for 06:00 peak | L2/L1 | banked (charge) |
| 06:00–08:00 | high (morning DHW + space) | discharging | run flat-out; heat consumed instantly | L2/L3 | free (discharge) |
| 08:00–17:00 | moderate space heat | tracking | run continuously; mix charge/discharge | L2 spot, L1 filler gaps | mostly free |
| 18:00–22:00 | high (evening peak) | discharging | run flat-out | L2/L3 | free (discharge) |
| 22:00–24:00 | declining | charge toward band top | run if job; else DR-charge | L1/L4 | banked |

**Day result (winter, demand-rich):** GPU runs ~24 h; thermal overlap ≈ 100% (all heat used or bankable); realized utilization = whatever compute demand the router + filler can supply (target ≥ 60%, defended by L1 filler so the floor holds even in spot droughts).

**Contrast — summer DHW-only day:** `E_day ≈ 4.5/365×... ≈ 6 kWh/day` of hot water, no space heat. Now `E_day (6) < GPU_max_daily (9.4)` ⇒ GPU **overproduces**. Sequence: run during/ahead of the two DHW draws (pre-heat the band, ~8.8 kWh), bank it, discharge into morning/evening use — but after the band saturates with no draw, further compute heat has nowhere to go. The GPU must **duty-cycle down** to ≈ `(6 + storage-cycled) / 9.4 ≈ 65–75%` of clock-hours on *thermal* grounds, and of the hours it does run, a chunk now pays retail electricity (overlap drops toward ~50%). The remaining idle hours route to **L4 DR** (summer peak-shaving is when VPP pays most) or coast. **This is the honest seasonal floor** the 60%×70% annual average is built to absorb — and the reason commercial (continuous draw, summer included) is the financeable case (§22, §28.2).

### 30.1 Utilization-engine interfaces & ownership

| Concern | Owned by | This Part's role |
|---|---|---|
| Thermal state, window, pre-heat plan | thermal coordinator (§6) | defines the model & math (§20–§21) |
| Job selection / yield arbitrage | yield router (§16) | defines the layer stack it routes (§25) |
| The interruptible filler backlog | §15 | specifies it as the utilization floor (§25 L1) |
| Reliability scoring | reliability service (§4) | defines the tiers it feeds (§24) |
| Checkpoint / resume / eviction | `connectivity-and-data.md` | specifies the preemption contract (§26) |
| Tank, sensors, GPU power-state, element | `node.md` | consumes its actuators & limits (§20, §29) |
| Seasonality risk to underwriting | `financial-and-roadmap.md` | produces the measured per-region model (§28) |
| Per-unit metering → securitization tape | §7 | writes every booked GPU-hour (§29 step 6) |

### 30.2 Utilization-engine open questions

- **`η_capture ≈ 0.95` is an unvalidated placeholder** (data-contracts §6) until real BOM/thermal data lands (`node.md`); it shifts the duty-cycle math materially and **every duty-cycle conclusion here is conditional on measuring it in Phase A**. (`C_th ≈ 0.352 kWh/°C` and `E_store ≈ 8.8 kWh` are *derived/canonical* per data-contracts §6, not placeholders, but are per-SKU and re-anchored from the registry as real tank specs land.)
- **Filler supply.** The floor guarantee (L1) only holds if there is *genuinely always* deadline-free backlog to run. Sourcing and sizing that backlog is a demand-layer (§15) problem the engine *depends* on; if backlog runs dry, `IDLE_AND_COAST` rises. Track filler-availability as a first-class fleet metric.
- **Residential overlap honesty.** §28 argues residential summer/warm-climate overlap is well below 70%. The per-region model must feed the tape with regional seasonal-weighted numbers, or residential pools will be over-underwritten (`financial-and-roadmap.md`).
- **DR vs comfort vs compute conflicts** are designed not to collide (DR only claims declined hours; comfort always preempts), but real utility DR contracts may impose firm shed obligations that fight a running premium L3 job. Reconcile DR contract terms with L3 SLAs before enrolling `premium` nodes in firm DR programs.

---

# Part D — Renter API & SDK

The OWN/DIRECT renter surface — the public REST/gRPC API, the SDK, and the `superheat` CLI that direct (non-marketplace) renters use to submit, observe, and pay for jobs. This is the Phase C ("own the demand & UX") revenue surface from `system-design-overview.md` §11.

**Authority:** This Part is a *consumer* of `data-contracts.md` (NORMATIVE). The external Job object defined here is a **projection** of the canonical `superheat.job.v1` (`data-contracts.md` §4). Where the two disagree, `data-contracts.md` wins; this surface must be updated, not forked.

> **How direct renters differ from marketplace renters.** Marketplace renters never see this API — they arrive through a connector (§12–§13) and their jobs are normalized into `superheat.job.v1` with `source ∈ {vast,runpod,…}`. Direct renters use *this* surface; their jobs enter with `source: "direct"`, `tenant_class: "direct"`, `platform_fee_frac: 0`, and reach the **same** yield router (§16) and scheduler (§5). This Part specifies only the external contract; the internals are unchanged.

## 31. The external Job object (a projection of `superheat.job.v1`)

A direct renter submits a **RenterJobSpec**. The control plane fills in the rest and produces the canonical `superheat.job.v1` (`data-contracts.md` §4). The renter never sets internal fields (`job_id`, `source`, `tenant_class`, the `economics` block, the resolved `net_price`). **Do not invent fields not derivable from the canonical schema.**

### 31.1 What the renter sends

```jsonc
// POST /v1/jobs  — RenterJobSpec (external projection)
{
  "name": "llama3-lora-run-7",            // renter-friendly label (maps to a tag, not job_id)
  "image": "cr.superheat/pytorch:2.4-cu124", // OCI ref; resolved to digest server-side
  "command": ["python", "train.py", "--epochs", "3"],
  "env": { "WANDB_API_KEY": "sk-…" },     // injected as container env (secrets ref preferred, §32.8)
  "resources": {
    "gpu_model": "RTX4090",               // canonical token (data-contracts §1); only RTX4090 GA today
    "gpu_count": 1,                        // 1..8 (>1 ⇒ commercial multi-GPU node)
    "min_vram_mb": 24000,                  // MB (data-contracts §1 units)
    "min_cpu_cores": 4,
    "min_ram_mb": 32000
  },
  "scheduling": {
    "interruptible": true,                 // false ⇒ non-interruptible (premium-tier, §34)
    "reliability_tier": "spot",            // {probation*,spot,standard,premium} (data-contracts §2)
    "expected_duration_s": 7200,           // SECONDS; null ⇒ open-ended (e.g. inference serving)
    "min_thermal_window_s": 1800,          // SECONDS of contiguous runnable time to start
    "max_start_latency_s": 300
  },
  "confidentiality": "standard",           // {standard, trusted_host} (data-contracts §2.2)
  "volumes": [
    { "name": "my-project", "mount": "/workspace" }   // persistent volume, §35.3 / connectivity-and-data.md
  ],
  "checkpoint": { "dir": "/scratch/ckpt", "interval_s": 900 },  // see connectivity-and-data.md
  "result_egress": { "mode": "renter_bucket", "uri": "s3://acme-out/run7/" }, // §36.2
  "idempotency_key": "acme-run7-2026-06-18" // dedups retried submits (§37.4)
}
```

> `reliability_tier: "probation"` is rejected at the API — probation nodes are filler/backlog-only and not directly rentable (`data-contracts.md` §2). Valid renter choices are `spot | standard | premium`.

### 31.2 Field mapping (external → canonical `superheat.job.v1`)

| External (this Part) | Canonical (`data-contracts.md` §4) | Notes |
|---|---|---|
| `image` | `compute.image_ref` | server resolves tag → `…@sha256:…` |
| `resources.gpu_model` | `compute.gpu_model` | canonical token `"RTX4090"` |
| `resources.gpu_count` | `compute.gpu_count` | 1..8 |
| `resources.min_vram_mb` | `compute.min_vram_mb` | MB |
| `scheduling.expected_duration_s` | `compute.expected_duration_s` | seconds; `null` ⇒ open-ended |
| `scheduling.interruptible` | `scheduling.interruptible` | gates window-fit predicate (`data-contracts.md` §7) |
| `scheduling.reliability_tier` | `scheduling.reliability_tier_required` | renter-chosen floor |
| `scheduling.min_thermal_window_s` | `scheduling.min_thermal_window_s` | seconds |
| `scheduling.max_start_latency_s` | `scheduling.max_start_latency_s` | seconds |
| `confidentiality` | `trust.confidentiality_required` | `standard`/`trusted_host` |
| *(server)* | `trust.needs_tee` | `true` only if `confidentiality=trusted_host` **and** a TEE-capable node requested; see §33 honesty |
| *(server)* | `source` = `"direct"` | always, for this surface |
| *(server)* | `tenant_class` = `"direct"` (or `"reserved"` under a contract) | |
| *(server)* | `economics.platform_fee_frac` = `0` | direct has no marketplace fee (§14) |
| *(server)* | `economics.gross/net_price_usd_per_gpu_hr` | from price quote (§32.6) or reserved contract |

Server-injected `trust.egress_policy` defaults to `"restricted"` and `trust.no_persistence` to `true` unless a persistent volume is attached (`connectivity-and-data.md`).

## 32. REST / gRPC API surface

Base URL `https://api.superheat.io`. gRPC mirror at `grpc.superheat.io:443` (service `superheat.renter.v1`); the wire schema is protobuf `superheat.job.v1` etc. per `data-contracts.md` §1 — JSON below is the human-readable projection. All times are UTC RFC-3339; all durations `_s` are seconds; money `_usd` is a decimal string.

### 32.1 Authentication

Two mechanisms; both yield the same `tenant_id` scope.

| Mechanism | Use | Header |
|---|---|---|
| **API key** | CI, SDK, CLI, scripts | `Authorization: Bearer sk_live_…` |
| **OAuth 2.0 (client-credentials)** | enterprise/SSO, short-lived tokens | `Authorization: Bearer <jwt>` |

- API keys are prefixed `sk_live_` / `sk_test_`, scoped (`jobs:write`, `jobs:read`, `volumes:*`, `billing:read`), and rotatable. The key body is shown **once** on creation.
- OAuth tokens carry `tenant_id`, `scopes`, `exp`; obtained from `POST /v1/oauth/token`.
- Every request must be over TLS 1.3. mTLS optional for enterprise tenants.

### 32.2 Endpoint table — jobs

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/jobs` | Submit a job (RenterJobSpec §31.1). Returns the external Job view. |
| `GET` | `/v1/jobs/{job_id}` | Get job status + lifecycle state (§34). |
| `GET` | `/v1/jobs` | List jobs (filter `state`, `name`, `created_after`; paginated §37.2). |
| `POST` | `/v1/jobs/{job_id}:cancel` | Request cancel; graceful checkpoint then stop (`data-contracts.md` §7 `T_GRACE_S=30`). |
| `GET` | `/v1/jobs/{job_id}/logs` | Stream/tail container logs (`?follow=true`, `?since=`, `?tail=`). |
| `POST` | `/v1/jobs/{job_id}/exec` | Open an interactive exec/PTY (WebSocket/gRPC stream) — backs `superheat ssh`. |
| `GET` | `/v1/jobs/{job_id}/events` | Per-job event stream (server-sent events / gRPC server stream). |

### 32.3 Endpoint table — volumes, quotes, billing, events

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/volumes` | Create a persistent volume (PoP-backed, content-addressed; `connectivity-and-data.md`). |
| `GET` | `/v1/volumes` | List volumes (name, size, `backed_by` PoP, status, `gc_in_s`). |
| `GET` | `/v1/volumes/{name}` | Volume detail incl. GC timers. |
| `DELETE` | `/v1/volumes/{name}` | Delete (key destruction + TRIM; `connectivity-and-data.md`). |
| `POST` | `/v1/quotes` | Price quote for a (tier × confidentiality × region × duration) request (§32.6). |
| `GET` | `/v1/usage` | Metered usage records (`superheat.usage.v1` projection; `data-contracts.md` §8). |
| `GET` | `/v1/billing/invoices` | Invoices / current-period running total. |
| `POST` | `/v1/webhooks` | Register a webhook endpoint + event filter (§32.7). |
| `GET`/`DELETE` | `/v1/webhooks/{id}` | Manage webhooks. |

### 32.4 Submit — example request/response

```http
POST /v1/jobs HTTP/1.1
Authorization: Bearer sk_live_…
Idempotency-Key: acme-run7-2026-06-18
Content-Type: application/json

{ …RenterJobSpec from §31.1… }
```

```jsonc
// 201 Created — external Job view
{
  "job_id": "sh_job_01HVICE9ZQK7",        // opaque external id (not the internal UUID)
  "name": "llama3-lora-run-7",
  "state": "queued",                       // §34 state machine
  "reliability_tier": "spot",              // ECHOED so renter knows what they got (§33)
  "confidentiality": "standard",
  "confidentiality_guarantee": {           // honest statement, see §33 & security-and-compliance.md
    "hardware_tee": false,
    "guarantees": ["encrypted_at_rest_per_job", "ephemeral_wipe_on_end", "encrypted_in_transit", "software_attestation"],
    "not_guaranteed": ["hardware_enforced_confidentiality_vs_host"]
  },
  "interruptible": true,
  "price": { "net_usd_per_gpu_hr": "0.38", "currency": "USD", "quote_id": "qt_…" },
  "created_at": "2026-06-18T14:03:22Z",
  "links": { "self": "/v1/jobs/sh_job_01HVICE9ZQK7", "logs": "…/logs", "events": "…/events" }
}
```

### 32.5 Status — example

```jsonc
// GET /v1/jobs/sh_job_01HVICE9ZQK7  → 200
{
  "job_id": "sh_job_01HVICE9ZQK7",
  "state": "running",
  "placement": { "region": "us-east-oh", "node_class": "H1", "gpu_count": 1 },
  "reliability_tier": "spot",
  "interruptible": true,
  "started_at": "2026-06-18T14:05:01Z",
  "preemptions": 1,                         // count so far (interruptible reality, §34)
  "last_checkpoint": { "step": 1200, "durable": true, "at": "2026-06-18T15:20:11Z" },
  "resume": { "rto_target_s": 300, "rpo_bound_s": 900 },  // connectivity-and-data.md
  "usage_running": { "gpu_seconds": 4500, "net_usd": "0.475" }
}
```

### 32.6 Price quote — example

```jsonc
// POST /v1/quotes
{ "gpu_model": "RTX4090", "gpu_count": 1, "reliability_tier": "standard",
  "confidentiality": "trusted_host", "region": "us-east-oh", "duration_s": 7200 }
```
```jsonc
// 200 — indicative, not a guaranteed reservation unless a reserved contract exists
{
  "quote_id": "qt_01HVID…",
  "net_usd_per_gpu_hr": "0.49",            // trusted-host + standard tier premium
  "breakdown": { "base_4090_usd_per_gpu_hr": "0.38",
                 "tier_premium": "0.06", "confidentiality_premium": "0.05" },
  "currency": "USD",
  "valid_until": "2026-06-18T14:18:22Z",   // short TTL; spot prices move (Part B / §16)
  "notes": "Indicative. Spot/interruptible price clears at placement; reserved contracts lock price."
}
```

Pricing rationale traces to Part B: direct demand carries `platform_fee_frac=0`, so the same gross is mostly net; `premium`/`trusted_host` carry premiums; `spot`/`interruptible` is the cheapest because it tolerates preemption.

### 32.7 Events & webhooks

Event types (same set on the per-job stream and webhooks): `job.queued`, `job.scheduled`, `job.running`, `job.preempted`, `job.resuming`, `job.checkpoint.durable`, `job.succeeded`, `job.failed`, `job.canceled`, `volume.gc_warning`, `volume.deleted`, `usage.record.sealed`.

```jsonc
// Webhook POST to the renter endpoint — signed
// Header: Superheat-Signature: t=1718719402,v1=hex(hmac_sha256(secret, t + "." + body))
{
  "id": "evt_01HVIE…",
  "type": "job.preempted",
  "created_at": "2026-06-18T15:31:00Z",
  "data": {
    "job_id": "sh_job_01HVICE9ZQK7",
    "reason": "thermal_window_closing",     // thermal_window_closing | higher_yield | node_fault | operator
    "durable_checkpoint": { "step": 1200, "at": "2026-06-18T15:20:11Z" },
    "expected_resume_by_s": 300              // RTO target (connectivity-and-data.md)
  }
}
```

Webhooks: at-least-once delivery, exponential backoff retry up to 24 h, HMAC-signed with a per-endpoint secret; renters must be idempotent on `event.id`.

### 32.8 Secrets

`POST /v1/secrets` stores a named secret (KMS-backed); reference it from `env` as `{ "WANDB_API_KEY": { "secret": "wandb_key" } }` so plaintext never transits in the job spec or appears in logs.

## 33. SDK / CLI contract

### 33.1 CLI verbs (idiomatic, mirrors `connectivity-and-data.md`)

| Command | Maps to | Purpose |
|---|---|---|
| `superheat run` | `POST /v1/jobs` | Submit + (optionally) attach. |
| `superheat ssh <job>` | `POST …/exec` | Interactive PTY over the brokered reverse tunnel (no inbound port; `connectivity-and-data.md`). |
| `superheat ls` | `GET /v1/jobs` / `…/volumes` | List jobs or volumes. |
| `superheat logs <job>` | `GET …/logs` | Tail/stream logs (`-f`, `--since`, `--tail`). |
| `superheat cp <src> <dst>` | volume/result transfer | Push inputs / pull results (PoP-mediated; never over a home uplink, `connectivity-and-data.md`). |
| `superheat ckpt <job>` | checkpoint control | Force a checkpoint now / list durable checkpoints (`connectivity-and-data.md`). |
| `superheat cancel <job>` | `POST …:cancel` | Graceful stop. |
| `superheat quote …` | `POST /v1/quotes` | Get a price quote. |
| `superheat volume {create,ls,rm}` | `/v1/volumes` | Manage persistent volumes. |

```bash
# Submit an interruptible LoRA run, attach to logs
$ superheat run --gpu RTX4090 --image cr.superheat/pytorch:2.4-cu124 \
      --tier spot --interruptible \
      --volume my-project:/workspace --ckpt-interval 900s \
      -- python train.py --epochs 3
job sh_job_01HVICE9ZQK7  tier=spot  conf=standard  net=$0.38/gpu-hr  state=queued

# tier & confidentiality are surfaced explicitly (see §33.3)
$ superheat ssh sh_job_01HVICE9ZQK7
renter@node:/workspace$    # a basement in Ohio; looks like a normal box

$ superheat ckpt sh_job_01HVICE9ZQK7 --now
checkpoint requested; durable step=1200 at 15:20:11Z (PoP us-east-oh)

$ superheat cp s3://acme-data/shard-*.tar my-project:/workspace/data/   # input → PoP, not home uplink
$ superheat cp sh_job_01HVICE9ZQK7:/workspace/out ./out/                # result via PoP down-link
```

### 33.2 Python SDK snippet

```python
import superheat as sh

client = sh.Client(api_key="sk_live_…")          # or OAuth via sh.Client.from_oauth(...)

job = client.jobs.run(
    image="cr.superheat/pytorch:2.4-cu124",
    command=["python", "train.py", "--epochs", "3"],
    gpu_model="RTX4090", gpu_count=1, min_vram_mb=24_000,
    tier="spot", interruptible=True,             # spot ⇒ cheap, preemptible (requires checkpointing)
    confidentiality="standard",
    volumes=[sh.Volume("my-project", mount="/workspace")],
    checkpoint=sh.Checkpoint(dir="/scratch/ckpt", interval_s=900),
    expected_duration_s=7_200,
    idempotency_key="acme-run7-2026-06-18",
)

# Honest tier/confidentiality disclosure (§33.3) — always available on the returned object
print(job.reliability_tier)               # "spot"
print(job.confidentiality.hardware_tee)   # False on a standard node
print(job.confidentiality.not_guaranteed) # ["hardware_enforced_confidentiality_vs_host"]

for ev in job.events():                   # server-stream: queued → running → preempted → resuming …
    if ev.type == "job.preempted":
        print("preempted:", ev.reason, "resume by ~", ev.expected_resume_by_s, "s")
    if ev.type in ("job.succeeded", "job.failed"):
        break

job.results.download("./out/")            # pulls from PoP staging or renter bucket (connectivity-and-data.md)
```

The SDK ships `superheat-ckpt` (`connectivity-and-data.md`): a thin PyTorch/HF/DeepSpeed callback that points the framework save path at `/scratch/ckpt/...` and triggers the async upload (stage 2). Using it is what upgrades a job from "best-effort restart" to "guaranteed cross-node resume."

### 33.3 How a renter is told their tier and confidentiality class (honest, no-TEE)

Every Job view echoes `reliability_tier` and a `confidentiality_guarantee` block (§32.4). The contract is deliberately explicit because the underlying hardware reality is unusual (`security-and-compliance.md`, `system-design-overview.md` §10).

| | `confidentiality: "standard"` (default) | `confidentiality: "trusted_host"` |
|---|---|---|
| Hardware | Residential RTX 4090 — **no robust hardware TEE** | Vetted commercial site / TEE-capable hardware |
| **Guarantees** | Per-job encryption-at-rest, ephemeral wipe-on-end, end-to-end encryption in transit, software attestation of node image/agent | All of standard **plus** vetted host + (where present) hardware confidential-compute |
| **Does NOT guarantee** | Hardware-enforced confidentiality of model/data **against a determined host** | (Reduced, hardware-dependent; still stated per node) |
| Price | Cheapest | Premium (§32.6) |
| Use for | Non-sensitive training/inference, batch ML, rendering | Sensitive models/data |

The SDK and CLI never silently downgrade: if a renter requests `trusted_host` and no eligible node exists, the job stays `queued` (or returns `409 no_capacity` on a quote), it is **never** placed on a `standard` node. This implements the `data-contracts.md` §7 feasibility rule: `confidentiality_required == "trusted_host"` ⇒ node must be `trusted_host`.

Reliability tier maps directly to `data-contracts.md` §2: `spot` = interruptible-only / cheapest; `standard` = reliable enough for most work; `premium` = non-interruptible-capable, high-uptime. A renter who omits `--interruptible` is requesting non-interruptible work, which forces `tier ≥ standard` (and usually `premium`) and a price premium.

## 34. Interruption UX

### 34.1 Job lifecycle state machine

```
                       submit (POST /v1/jobs)
                                │
                                ▼
        ┌───────────────────► QUEUED ──────────── cancel ──────────► CANCELED
        │  (no eligible node /     │
        │   no capacity yet)       │ scheduler places (data-contracts §7 fits())
        │                          ▼
        │                      SCHEDULED ───────── start fails ─────► FAILED
        │  (window reopens,        │
        │   resume target)         │ container starts, weights pulled (connectivity-and-data.md)
        │                          ▼
        │       ┌──────────────► RUNNING ─────── exit 0 / completion ─► SUCCEEDED
        │       │  resume @          │  │
        │       │  durable step      │  │ exit≠0 / OOM / image error ─► FAILED
        │       │ (connectivity-…)   │  │
        │       │                    │  │ cancel (graceful, T_GRACE_S=30)─► CANCELED
        │   RESUMING ◄── reassign ── │  │
        │       ▲                    ▼  │
        └───────┴──────────────── PREEMPTED
                (interruptible only:  thermal_window_closing |
                 higher_yield | node_fault | operator)
```

This is the renter-facing projection of the internal job lifecycle (§5.3). The internal preemption mechanics that drive these transitions are Part C / §26.

- **Non-interruptible jobs never enter `PREEMPTED`** — they only run on a window that fits the whole `expected_duration_s` (`data-contracts.md` §7), so a closing window cannot interrupt them.
- **Interruptible jobs** cycle `RUNNING → PREEMPTED → RESUMING → RUNNING` an unbounded number of times; `preemptions` is surfaced in status (§32.5) and each transition emits `job.preempted` / `job.resuming` events.

### 34.2 How preemption is surfaced + checkpoint/resume contract (renter POV)

- A preemption is **never** a failure for a checkpointed job — it is a `job.preempted` event followed by `job.resuming` then `job.running`. From `superheat ssh`/`exec`, the brokered tunnel re-points to the new node (`connectivity-and-data.md`), so the endpoint URL is stable; the renter sees a brief stall.
- **Checkpoint contract (renter POV):** the renter writes checkpoints to `checkpoint.dir` (default `/scratch/ckpt`) using `superheat-ckpt` or a framework hook. The system writes local NVMe (fast), uploads deltas to the PoP asynchronously during compute, and registers durability (`connectivity-and-data.md`). The job is "safely resumable up to step n" only after the PoP-durable registration of step n.
- **What resume guarantees:** RTO ≤ 5 min (p50 ≤ 2 min, p95 ≤ 10 min) for working sets ≤ 40 GB; RPO ≤ one checkpoint interval (default ≤ 15 min of compute) — `connectivity-and-data.md`. The in-flight 30 s grace flush (`data-contracts.md` §7 `T_GRACE_S`) cannot finish a multi-minute upload, so the **durable resume source is always the last PoP-persisted checkpoint**, not the in-flight flush.
- **Opaque jobs honesty:** a job that does not checkpoint (no SDK/framework hook) gets best-effort CRIU/`cuda-checkpoint` and may restart from scratch on preemption (`connectivity-and-data.md`). The SDK warns on submit; such jobs should choose `tier ≥ standard` non-interruptible or accept the restart risk.

### 34.3 Persistent volumes & eviction semantics

Per `connectivity-and-data.md`, the renter sees four storage classes with published lifetimes (configurable per contract/tier):

| Class | Survives preempt? | Survives job end? | Renter backs up? | Default GC |
|---|---|---|---|---|
| Ephemeral scratch | No | No | n/a (disposable) | on node change |
| Checkpoint dir | Yes (via PoP) | retained then GC'd | no (auto) | `T_ckpt` = 24 h after job end |
| Persistent volume (`--volume`) | Yes (re-attached) | retained then GC'd | yes, before expiry | `T_vol` = 7 d after last detach |
| Result/output | Yes | retained then GC'd | pulled or auto-pushed | `T_out` = 72 h in PoP |

`volume.gc_warning` events fire at 48 h and 6 h before `T_vol`. `superheat volume ls` shows `GC-IN` so eviction is never a surprise.

### 34.4 SLO / SLA per tier

| Tier (`data-contracts.md` §2) | Interruptible? | Start-latency SLO (p95) | Availability SLO during a placed window | Preemption | Resume RTO / RPO | Credits |
|---|---|---|---|---|---|---|
| `premium` | No (or opt) | ≤ 60 s | ≥ 99.0% of contracted window | None within window (whole job fit before start) | n/a (no preempt) | SLA-backed (reserved contract) |
| `standard` | Optional | ≤ 180 s | ≥ 97% per placed window | Rare; checkpointed | RTO p95 ≤ 10 min / RPO ≤ 15 min | Best-effort credits |
| `spot` | Yes (only) | ≤ 300 s (`max_start_latency_s`) | Best-effort | Expected; can preempt any time window closes or yield wins | RTO p95 ≤ 10 min / RPO ≤ 15 min | None (priced for it) |

SLOs are operational targets; SLAs (with credits) attach only to `premium`/reserved contracts (§14 reserved-floor). Spot is explicitly best-effort — that is what makes it the cheapest tier.

## 35. Data I/O from the renter side

### 35.1 Pushing images & datasets

- **Images:** reference any OCI image (`cr.superheat/...`, `ghcr.io/...`, etc.). On first use a region pulls each layer **once** to the PoP cache (`connectivity-and-data.md`, content-addressed dedup); subsequent jobs hit cache. Renters never push images over a home uplink.
- **Datasets / inputs:** `superheat cp` or `client.volumes.put()` uploads to the **PoP/origin once**, not to the node. The node pulls down (fast direction) at job start. A 40 GB dataset to a 1 Gbps PoP backhaul is ~7 min vs ~5.6 h over a 20 Mbps home uplink (`connectivity-and-data.md`). Prefer referencing public artifacts (HF models, registry images) — those are already cache-warmed.

### 35.2 Pulling results

Two modes (`connectivity-and-data.md`), set via `result_egress`:
1. **`renter_bucket` (preferred):** supply scoped write-only S3/GCS creds (or a secret ref); the node pushes results directly to your bucket over a direct mesh path, rate-limited below the home `up_mbps_floor_24h` so it never saturates the homeowner link. Keeps large outputs off Superheat infra.
2. **`pop_staging`:** results land in PoP staging (`T_out` = 72 h), pulled via `superheat cp <job>:/path ./local` over the fast PoP down-link.

### 35.3 Regional cache & bandwidth realities

- The renter benefits transparently from the regional content-addressed cache; no API call required. `region` can be pinned in `placement_hints` to co-locate with a dataset's PoP.
- **Bandwidth reality the renter must know:** the home **uplink** is the constraint (often 10–20 Mbps). Big-checkpoint / big-output jobs are flagged `data_heavy` and steered to A-tier-uplink nodes (`connectivity-and-data.md`); a renter requesting huge result egress on a `spot` job may see longer effective egress. The SDK exposes `job.placement.bandwidth_tier` so this is observable.

## 36. (folded into §35)

*Data I/O from the renter side is §35; the PoP-mediated push/pull and bandwidth model it projects are owned by `connectivity-and-data.md`. (`result_egress` modes are §35.2.)*

## 37. Rate limits, quotas, errors, idempotency, versioning

### 37.1 Rate limits & quotas

| Resource | Default limit | Header / enforcement |
|---|---|---|
| API requests | 600 req/min/tenant (burst 1200) | `429`, `Retry-After`, `X-RateLimit-Remaining` |
| Concurrent running GPUs | tenant quota (default 16; raise via support / contract) | `409 quota_exceeded` on submit |
| Jobs created | 120/min/tenant | `429` |
| Log/exec streams | 20 concurrent/tenant | `429` |
| Webhook deliveries | provider-side; backoff on renter `5xx` | retry up to 24 h |

### 37.2 Pagination

List endpoints use opaque cursors: `?limit=50&cursor=…`; response carries `{ "items": [...], "next_cursor": "…"|null }`. Max `limit` 200.

### 37.3 Error model

RFC-9457 problem+json. Stable machine-readable `code`; never parse `detail`.

```jsonc
// 409 Conflict
{
  "type": "https://docs.superheat.io/errors/no_capacity",
  "code": "no_capacity",
  "title": "No eligible node for the requested tier/confidentiality",
  "status": 409,
  "detail": "No trusted_host RTX4090 node available in us-east-oh; job remains queued.",
  "request_id": "req_01HVIF…",
  "retryable": true
}
```

| `code` | HTTP | Meaning |
|---|---|---|
| `invalid_argument` | 400 | Spec failed validation (e.g. `gpu_count` out of 1..8, unit mismatch). |
| `unauthenticated` | 401 | Missing/invalid key or token. |
| `permission_denied` | 403 | Key lacks scope. |
| `not_found` | 404 | Unknown `job_id`/volume. |
| `conflict` | 409 | Idempotency-key reuse with different body; or `quota_exceeded`; or `no_capacity`. |
| `tier_not_rentable` | 422 | `reliability_tier: "probation"` requested (not directly rentable). |
| `confidentiality_unsatisfiable` | 422 | `trusted_host` requested but disallowed by region/policy. |
| `rate_limited` | 429 | See `Retry-After`. |
| `internal` | 500 | Server error; retry with backoff. |

### 37.4 Idempotency

- Mutating POSTs accept an `Idempotency-Key` header (or `idempotency_key` in body). The server stores the first response for 24 h and returns it verbatim on retry with the same key + same body. Same key + **different** body ⇒ `409 conflict`.
- This makes `superheat run` and the SDK safe to retry on network failure without double-submitting a job.

### 37.5 Versioning

- URI-versioned (`/v1/...`); the wire protobuf is `superheat.<thing>.v<major>` (`data-contracts.md` §1). The external Job is a projection of `superheat.job.v1`; a canonical breaking change (new `vN`) is mirrored by a new API major.
- Additive fields are non-breaking; clients must ignore unknown fields. Deprecations announced via `Sunset` header + ≥ 90-day window. The current canonical schema version is echoed in every Job view as `schema: "superheat.job.v1"`.

### 37.6 How the renter surface ties into the system

| Concern | This Part provides (external) | Backed by |
|---|---|---|
| What a direct renter submits | RenterJobSpec → projection of `superheat.job.v1` | `data-contracts.md` §4 |
| Routing/pricing of direct jobs | `source:"direct"`, fee=0, into the yield router | Part B (§14, §16) |
| Tier & confidentiality semantics | Echoed tier + honest no-TEE confidentiality block | `data-contracts.md` §2/§2.2, `security-and-compliance.md` |
| Interruption / checkpoint / resume | Lifecycle states, events, RTO/RPO, volume eviction | `connectivity-and-data.md`; internal mechanics Part C / §26 |
| Data movement | PoP-mediated push/pull, bandwidth realities | `connectivity-and-data.md` |
| Metering / billing | `/v1/usage`, invoices | `superheat.usage.v1` (`data-contracts.md` §8), Part A / §7 |

**One-line takeaway:** the direct renter surface is a thin, honest projection of the canonical job model — it lets a customer rent a basement RTX 4090 with data-center ergonomics (one command, stable session, clear preemption + eviction semantics) while being explicit about the two things that are genuinely different: interruptibility and the absence of hardware-enforced confidentiality on standard nodes.

---

## Cross-references

- `data-contracts.md` — NORMATIVE canonical schemas: job `superheat.job.v1` (§4), telemetry `superheat.telemetry.v1` (§5), usage `superheat.usage.v1` (§8), reliability score/tier (§2/§3), confidentiality axis (§2.2), units (§1), tank/thermal constants (§6), feasibility `fits()` + `T_GRACE_S` (§7), scheduler projection (§9).
- `system-design-overview.md` — the five principles and the headline economics this platform implements.
- `node.md` — on-node agent + immutable OS, the 4090 SKU, the 3.3 kW thermal system, tank/sensors/GPU power-state/element actuators consumed by the thermal coordinator and utilization engine.
- `connectivity-and-data.md` — outbound-only WireGuard overlay; regional caches, checkpoint/resume, persistent volumes, eviction, bandwidth-aware placement, the CLI/`superheat-ckpt` data path.
- `security-and-compliance.md` — attestation, tenant isolation, key management, no-TEE honesty, the `trusted_host` confidentiality class, wipe-on-end; legal/regulatory.
- `financial-and-roadmap.md` — CapEx/OpEx and the ABS/securitization model the metering ledger feeds, the 60% concentration covenant, build phasing, the seasonality and concentration risk registers.
- `operations.md` — observability/SLO/incident response, testing/simulation, field operations & onboarding.
