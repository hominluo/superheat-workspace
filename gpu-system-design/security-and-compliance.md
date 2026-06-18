# Superheat — Security, Trust & Compliance

**Status:** Draft v0.1 · **Date:** June 2026 · **Audience:** Engineering + security reviewers · Founders, ops/field leads, finance/structuring counsel, partner (installer) network · financing/diligence
**Scope:** This is the authoritative design and obligations document for how Superheat keeps three mutually-distrusting parties safe from each other on a single GPU node — and for the regulatory, contractual, and insurance obligations Superheat must satisfy to legally **install** a GPU-in-a-water-heater on a third party's premises, **rent** the compute, **finance** the unit at point of sale, and **securitize** the cash flows. It expands the "Security, multi-tenancy & trust" section of `system-design-overview.md`, and frames compliance so it gates an install or a securitization rather than surprising us.

- **Part A — Security, multi-tenancy isolation & trust:** how three parties stay safe on one node, what we can and cannot guarantee, and the operational controls that make the fleet operable by a third party if Superheat fails (a securitization requirement from `../market-research/Superheat_Compute-Network_Debt-Engine_Onepager.md`).
- **Part B — Legal, regulatory, compliance & insurance:** an engineering/ops framing of *what we must satisfy*, *who owns it*, and *what artifact proves it*. Every obligation has a named owner and a **gating artifact** — a document, certificate, or signed agreement that must exist before the gated action proceeds.

> **Posture in one line.** Superheat operates a **regulated physical appliance** (electrical + plumbing + appliance-safety law), a **multi-tenant compute service** on **third-party residential premises** (isolation + data-protection + lawful-process + abuse law), and a **consumer-finance + securitization** business (lending + securities law) — three regulatory worlds at once, multiplied by every jurisdiction. The unifying controls are **fail-safe-to-heater**, **honest tiering of what we guarantee**, and **per-region gating** (Part B §15).

**This is not legal advice.** Part B is an engineering/ops framing; every obligation owned by legal/finance/tax counsel must be confirmed with qualified counsel in each jurisdiction before deployment, financing, or marketing.

**Two gates Part B exists to protect:**
- **The install gate** — what must be true before a unit is energized for compute on a customer's premises.
- **The securitization gate** — what must be true before a unit (and its revenue) can enter a financed pool / the ABS tape.

Cross-references: `node.md` (immutable OS, node agent, attestation root, electrical/3-phase, UL, fail-safe interlock, year-5 GPU swap), `connectivity-and-data.md` (outbound-only overlay, egress firewall, key management, ephemeral storage, wipe-on-job-end, caching, regional caches), `platform.md` (registry, scheduler/yield router, feasibility predicate, region tagging, audit ledger, metering/billing, demand routing, renter API/SLA), `data-contracts.md` (confidentiality_class, reliability_tier, double-signed usage records, reconciliation), `financial-and-roadmap.md` (phasing, residual risk register, insurance OPEX, payout/sanctions rails, sales-tax/ITC into the tape), `operations.md` (installer certification, per-install permit/inspection runbook, incident response), `system-design-overview.md` (design principles, pooling/seasoning).

---

# Part A — Security, Multi-Tenancy Isolation & Trust

## 1. The problem: three mutually-distrusting parties on every node

A Superheat node is not a data-center server. It is an RTX 4090 embedded in a water heater, **in someone else's home or building**, behind that person's residential internet, running a stranger's code. On every single node there are three parties who do not trust each other and should not have to:

| Party | What they bring | What they fear | What they must never get |
|---|---|---|---|
| **Renter** | Untrusted code + data + model weights. Wants compute and, ideally, confidentiality. | Host or operator stealing their model/data; another tenant interfering with their job. | Access to the host LAN, the host's data, the thermal controller, or other tenants. |
| **Homeowner / host** | The physical site, the LAN, the electricity, and — critically — their hot-water service and privacy. | A renter attacking their home network or devices; losing hot water; being implicated in a renter's illegal use. | Nothing from the renter's workload; no liability for renter behavior. |
| **Superheat (operator)** | The hardware, the signed software, the control plane, the metering ledger. | A compromised node poisoning the fleet or the tape; reputational/legal exposure from abuse. | (Operator is the trust anchor, but is itself partially distrusted by both other parties — see §4 and §5.) |

The five design principles from `system-design-overview.md` constrain every control below:

1. **Outbound-only** — no inbound ports on the home router, so no inbound attack surface and no renter-reachable listener on the host LAN.
2. **Fail-safe-to-heater** — no security control may ever leave a homeowner without hot water; isolation failure modes degrade to "compute off, heater on."
3. **Compute thermally coupled** — the thermal controller is a safety-critical device and is on the *other* side of every isolation boundary from the tenant.
4. **Measure-then-price reliability** — security/uptime evidence is metered per node and feeds both routing and the financing tape (§7).
5. **Hybrid demand** — tenants arrive via marketplaces *and* our own API; isolation and tier labeling must hold identically regardless of demand source.

---

## 2. Threat model

### 2.1 Trust boundaries

```
   UNTRUSTED                    SEMI-TRUSTED HOST            TRUST ANCHOR
 ┌───────────┐   isolation   ┌──────────────────┐   attest  ┌─────────────┐
 │  RENTER   │──boundary 1──▶│  HOMEOWNER / LAN  │──tunnel──▶│  SUPERHEAT  │
 │  workload │               │  + host OS + tank │           │ control     │
 └───────────┘               └──────────────────┘           │ plane       │
       │  isolation                  │  must never              └─────────────┘
       │  boundary 2                 │  harm hot water
       ▼                             ▼
 ┌───────────┐               ┌──────────────────┐
 │  OTHER    │               │ THERMAL          │
 │  TENANTS  │               │ CONTROLLER (safe) │
 └───────────┘               └──────────────────┘
```

### 2.2 Threat-model table (actor → threat → control)

| # | Actor | Threat | Primary control(s) | Ref |
|---|---|---|---|---|
| T1 | Renter | Pivot from container to **host LAN** (scan/attack homeowner's devices: cameras, NAS, IoT, router admin) | Default-deny egress firewall; RFC1918 + link-local + host-subnet blackholed at node; no bridged/host networking | §3.4, `connectivity-and-data.md` |
| T2 | Renter | Escape container to **host OS / root** | microVM boundary for untrusted tiers (§3.2); seccomp/AppArmor + dropped caps + userns for containers; read-only immutable host root | §3, `node.md` |
| T3 | Renter | Reach the **thermal controller** and disable/abuse heating or overheat the tank | Thermal controller on a physically/logically separate bus; node agent is sole writer; tenant has no route or device node to it; hard local thermal limits in firmware | §3.5, `node.md` |
| T4 | Renter | **Tenant→tenant** interference on multi-GPU commercial nodes (memory, GPU state, side channels, noisy-neighbor) | One tenant per GPU (no MPS sharing across tenants by default); per-tenant cgroups + GPU VRAM clear between jobs; separate microVMs | §3.3 |
| T5 | Renter | **Abusive workload**: crypto mining | Behavioral + telemetry mining detection; ToS auto-enforcement; throttle/kill/ban | §5 |
| T6 | Renter | **Abusive workload**: DDoS source / spam / port-scan from the node | Egress rate limits, connection-rate caps, blocked SMTP/known-abuse ports, anomaly detection | §3.4, §5, `connectivity-and-data.md` |
| T7 | Renter | **Illegal content** generation/storage/distribution (CSAM, etc.) | KYC on **direct renters only** (does not transfer to marketplace-sourced renters — detection-and-kill there, §5.2); no-persistence default; abuse-report pipeline; rapid kill + preservation for law enforcement; legal-hold path | §5, Part B §11 |
| T8 | Homeowner / host | **Snoop renter data/model** by reading RAM, VRAM, disk, PCIe, or DMA on hardware they physically own | **HONEST LIMIT (§4): cannot be fully prevented on a standard 4090 node.** Mitigations: ephemeral, encrypted-at-rest, no-persistence, wipe-on-job-end; trusted-host tier for sensitive work | §4 |
| T9 | Homeowner / host | Tamper with node software to **inflate metered hours** or run unauthorized work | Secure boot + measured boot + remote attestation before paid work; node identity key in TPM/secure element where available | §4, §6 |
| T10 | Homeowner / host | **Deny/throttle** the node to harm a renter or game payouts | Reliability scoring detects; no payout for unattested/under-delivering nodes; renter SLA tied to tier | §6, `platform.md` |
| T11 | External attacker | Compromise the **home router/LAN** and pivot into the node | Outbound-only (no inbound surface); node treats the host LAN as hostile by default; mutually-authenticated tunnel; node does not trust LAN-sourced traffic | `connectivity-and-data.md` |
| T12 | External attacker | **Impersonate the control plane** or MITM the tunnel | Mutual auth (WireGuard static keys + cert-pinned control API); signed commands; key rotation/revocation | §6, `connectivity-and-data.md` |
| T13 | Compromised node | A rooted/cloned node **poisons the fleet**: fake telemetry, malicious OTA acceptance, ledger fraud | Attestation gating, signed OTA only (§6), per-node anomaly detection, quarantine + revoke; node cannot sign for other nodes | §6, §5 |
| T14 | Compromised node | A node exfiltrates **other tenants'** or fleet secrets | No shared fleet-wide secrets on nodes; per-node least-privilege keys; secrets scoped to the single node + job | §6 |
| T15 | Insider (Superheat) | Operator staff misuse access to renter data or node control | Least-privilege RBAC, audit logging of all operator actions, break-glass with review, escrowed runbooks (§7) | §7, `platform.md` |

### 2.3 Explicitly out of scope (and why)

- **Defeating a determined host with physical hardware access on a standard node.** Stated plainly in §4. Cold-boot/DMA/bus-probing attacks against a consumer 4090 cannot be cryptographically prevented; we manage this by data hygiene + tiering, not by pretending otherwise.
- **Covert hardware implants installed by the host before deployment.** Mitigated operationally for the trusted-host tier (vetted sites, sealed units); accepted residual risk on standard nodes.
- **Nation-state targeting of a specific renter's model.** Out of scope for the standard tier; such renters belong on the trusted-host tier or off-network.

---

## 3. Tenant isolation architecture

### 3.1 Goal

Enforce four boundaries on every node, with **fail-safe-to-heater** as the overriding constraint: if any isolation component fails to start or is unhealthy, the node **refuses to run tenant work and continues operating as an ordinary water heater** (the recovery ladder in `node.md`).

1. **Tenant → host OS** (no escape to root / immutable base)
2. **Tenant → home LAN** (no lateral movement into the homeowner's network)
3. **Tenant → tenant** (no cross-tenant access on multi-GPU nodes)
4. **Tenant → thermal controller** (no access to the safety-critical heating path)

### 3.2 Containers vs microVMs on consumer GPUs — the real tradeoff

The hard constraint: **GPU passthrough into a strongly-isolated sandbox is mature for containers and immature/heavier for microVMs on consumer cards.**

| Approach | GPU access path | Isolation strength | Cost on a 4090 node | Verdict |
|---|---|---|---|---|
| **NVIDIA Container Toolkit (containers)** | Native, well-supported; renters expect this UX (matches Vast.ai/RunPod) | Kernel-shared; relies on namespaces + seccomp + AppArmor + cap-drop + userns. Escape surface = host kernel. | Lowest overhead; near-native GPU perf | **Default for the workhorse path**, hardened (see §3.3) |
| **Firecracker microVM** | No first-class consumer-GPU passthrough; VFIO GPU passthrough is not Firecracker's model | Strong (separate kernel) for CPU/mem | GPU passthrough effectively unavailable → unusable for GPU jobs today | Not for GPU work now; track in `financial-and-roadmap.md` |
| **Cloud Hypervisor + VFIO GPU passthrough** | VFIO passthrough of the whole 4090 into a VM is feasible | Strong (separate kernel + IOMMU-isolated device) | Heavier; one GPU bound to one VM (no fractional sharing); driver/IOMMU complexity on consumer boards | **Candidate for high-isolation / trusted-host tier and multi-tenant commercial nodes** |
| **Kata Containers** | OCI UX over a microVM; consumer-GPU passthrough still constrained by the same VFIO realities | Strong | Medium-heavy; container ergonomics | Track as the convergence path (microVM strength + container UX) |

**Decision:**
- **Standard tier / home H1 nodes (single tenant per node):** hardened **containers via NVIDIA Container Toolkit**. Because an H1 is single-tenant, the tenant↔tenant boundary is moot, and the dominant boundaries (tenant→LAN, tenant→thermal) are enforced at the *network and device* layer (§3.4, §3.5), not only at the container boundary — so a kernel-shared container is acceptable here.
- **Multi-GPU commercial nodes (multi-tenant):** prefer **VFIO GPU passthrough into a per-tenant VM (Cloud Hypervisor / Kata)** so a container escape on one tenant cannot reach another tenant's GPU/memory. Where passthrough is not yet stable on a given board, fall back to hardened containers with **one tenant per GPU and full VRAM clear between jobs**, and label the node accordingly in the registry.
- **Trusted-host tier:** VM-per-tenant is the baseline; this is also where future TEE-capable hardware (§4.4) plugs in.

This layering is consistent with `node.md`: the host is an immutable/atomic Linux base with a read-only root and signed images only, so even a container escape lands in a minimal, non-persistent, signed environment rather than a general-purpose box.

### 3.3 Hardening the container path (when containers are used)

- **Rootless / user-namespaced** containers; tenant root maps to an unprivileged host uid.
- **Drop all capabilities**, add back none beyond what the GPU runtime strictly needs; `no-new-privileges`.
- **seccomp** profile (deny exotic syscalls) + **AppArmor/SELinux** confinement.
- **Per-tenant cgroups v2**: CPU, RAM, PIDs, block-I/O, and device allowlist (only the assigned `/dev/nvidia*` nodes; **never** the thermal device, never raw block devices, never the NIC in host mode).
- **No host networking, no privileged mode, no docker.sock**, no host bind-mounts except an explicit ephemeral scratch (`connectivity-and-data.md`).
- **One tenant per GPU.** MPS/MIG-style sharing is **not** used across distinct tenants by default; MIG is unavailable on the 4090 and time-slicing leaks side-channel + noisy-neighbor risk.
- **GPU VRAM scrub between jobs** (allocate-and-zero or driver reset) so the next tenant cannot read residual weights/activations. **Honest limitation:** complete VRAM clearing on consumer 4090s is a **partially-open problem**, not a fully solved guarantee — published research shows memory residue can persist **across CUDA contexts** on consumer NVIDIA cards (allocate-and-zero only covers regions we successfully reclaim; a driver/GPU reset is more thorough but not always available or clean on consumer boards). We treat residual-VRAM leakage to the next tenant as a residual risk (added to §4.5 honest limitations), mitigated but not eliminated by one-tenant-per-GPU and best-effort scrub, and we do not claim it is fully guaranteed.

### 3.4 Per-tenant egress firewall (default-deny) — coordinates with `connectivity-and-data.md`

The single most important homeowner-protection control. The renter's network namespace gets a **dedicated egress path that never touches the host LAN.**

- **Tenant traffic is policy-routed into the Superheat overlay only.** The renter container/VM has **no route to the host's LAN subnet, the default gateway's LAN side, link-local (169.254/16), multicast/mDNS, or RFC1918 ranges** that belong to the home. Those destinations are dropped at the node.
- **Default-deny egress.** Outbound is blocked except: (a) the overlay control/data endpoints, (b) an allowlisted set of common model/dataset/registry/object-store destinations (or all-internet for tiers/renters that opt in and accept abuse monitoring), (c) DNS via a Superheat resolver (no host DNS, no DNS to the home router).
- **Anti-abuse rules baked in:** block outbound SMTP (25/465/587) by default, cap new-connection rate and total egress bandwidth per tenant, block known-mining-pool ranges/ports, and flag scan-like fan-out patterns (T6).
- **No inbound** ever (outbound-only principle): no listener on the renter side is reachable from the home LAN or the internet. Renter "ingress" (SSH/console) is mediated **through the overlay**, terminated by the node agent, never via a home-router port.

The net guarantee to the homeowner: **a renter's packets cannot be addressed to anything on the home network.** Combined with §3.5, this is how we promise the host that the renter can never harm their network, devices, or hot water (§6 host-protection summary). This isolation is also load-bearing for host indemnification (§5.4) and for keeping the homeowner out of legal process aimed at a renter (Part B §11.3).

### 3.5 Protecting the thermal controller (tenant → safety device)

- The thermal controller (reads tank temp / heat demand, controls GPU power state per `system-design-overview.md`) is reachable **only by the node agent**, over a separate interface (dedicated serial/I²C/GPIO or a separate microcontroller), never exposed as a device the tenant cgroup can open.
- **Local safety always wins over remote scheduling and over any tenant behavior.** Hard thermal/over-temp limits live in the controller firmware and the node agent's local safety loop; no tenant workload (or even the control plane) can override them. This hardware safety interlock is independent of the host OS and the cloud, which is what makes it a *certifiable* claim (Part B §10.2).
- A tenant maxing the GPU only produces *more useful heat or banked heat* (the tank is a thermal battery, per the overview) or triggers a throttle — it can never disable heating or unsafely overheat the tank.

### 3.6 Multi-GPU commercial node specifics

- Tenants are partitioned at **GPU granularity**; the host NIC, storage controller, and thermal bus are never assigned to a tenant.
- Prefer **VM-per-tenant with VFIO** (§3.2) so the tenant↔tenant boundary is a hypervisor + IOMMU boundary, not just a namespace.
- Shared NVMe scratch is partitioned per-tenant with separate encryption keys (`connectivity-and-data.md`); no shared writable mount.

---

## 4. The honest confidentiality limitation (read this plainly)

### 4.1 The statement we make to every renter

> **On a STANDARD Superheat node, we cannot provide hardware-enforced confidentiality of your data or model against a determined host.** Consumer RTX 4090s do not have the robust, attestable hardware TEE / confidential-computing capability that data-center GPUs (e.g., H100 with Confidential Computing) increasingly provide. The person who owns the building also owns the silicon, the RAM, and the PCIe bus. A motivated host with physical access can, in principle, read system RAM, GPU VRAM, or bus traffic. **We do not claim otherwise.** If your workload requires that guarantee, use the **Trusted-Host tier** (§4.3) or do not run it on standard nodes.

This honesty is a feature, not a weakness: it is what makes the financing tape credible (`financial-and-roadmap.md`) and keeps Superheat out of the position of having sold a confidentiality promise it cannot keep. It also carries a **legal edge** (Part B §10.4): representing standard nodes as confidential would be a misrepresentation/UDAP risk, and regulated renter data belongs on the `trusted_host` class or off-network.

### 4.2 What we therefore DO protect against (software/curious-host threats)

Most real-world "snooping" is opportunistic, not a hardware lab attack. Against the **curious or casual host** and against **data-at-rest leakage**, we apply defense-in-depth (coordinated with `connectivity-and-data.md`):

- **Ephemeral by default.** Tenant workloads run in throwaway containers/VMs; nothing persists across jobs unless the renter explicitly buys a persistent volume.
- **Encrypted-at-rest, with a single canonical per-job key lifecycle.** Tenant scratch and any persistent volume are encrypted with a **per-job data-encryption key** whose lifecycle is defined once here (and is the only correct description — earlier drafts described it three inconsistent ways: node-memory-only, control-plane-escrowed, and KMS-never-on-node; the contradiction is resolved as follows):
  - **Derivation:** the key is generated/held by the **control plane** (its KMS), scoped to a single `job_id`. It is never derived on or by the node.
  - **Delivery:** it is delivered to the node **over the authenticated, encrypted overlay** (`connectivity-and-data.md`) at job start, into the node agent's memory only.
  - **Residency:** it lives **only in node RAM for the duration of the job**. It is **never persisted to node disk** (no swap, no on-disk copy) and the host's own persistent partitions never see it.
  - **Drop:** on job completion/preemption the node agent **drops the key from memory** and the control plane retires its copy for that `job_id`. Dropping the key is the **cryptographic erase** of all at-rest data for that job.
  - **Control-plane-compromise blast radius:** because the KMS holds per-job keys, a compromise of the control-plane KMS exposes the at-rest ciphertext of jobs whose keys it still holds (live + not-yet-retired) — it does **not** expose jobs whose keys have already been dropped (forward-secret against the at-rest store). It never exposes a fleet-wide master that decrypts everything, and keys are not shared across nodes or jobs.
  - **Honest residual:** the key, and therefore the data, is **in cleartext in RAM (and VRAM) on the node during compute** — by necessity, to run the job. A root host can read it there (see the live-memory statement below and §4.1). Encryption-at-rest only protects the data **at rest**, not in use.
- **No-persistence-by-default + wipe-on-job-end.** On job completion/preemption: kill the sandbox, **drop the per-job key** (cryptographic erase, per the lifecycle above), scrub VRAM (§3.3 — partial; see limits), and TRIM/zero the scratch region. A host who later images the disk finds ciphertext with a dropped key.
- **In-transit encryption.** All data to/from the node rides the authenticated, encrypted overlay (`connectivity-and-data.md`); weights/datasets come from regional caches over that channel, not from the host.
- **Minimize residency.** Pre-stage only what the job needs; stream where possible; never persist renter inputs longer than the job.

**What these controls do and do NOT cover — stated plainly:** encryption-at-rest + wipe-on-job-end defend against **post-hoc disk forensics** (a host who images the NVMe after the job) and **casual/opportunistic snooping**. They do **NOT** protect a renter's data or model from a host who reads **live RAM or VRAM during execution**. To compute on the data, it must be decrypted into memory; a host with root on the box can read process memory and GPU memory with standard tools (e.g. `/proc/<pid>/mem`, a debugger, `nvidia-smi`/CUDA debug tooling, or DMA). Encryption-at-rest does nothing once the bytes are live. So these controls raise the bar from "trivial for any host" to "requires effort, and only after the job is over" — but against **live memory during the job they offer essentially no protection**, and they are **not** a substitute for a TEE. We say so.

Note also that **software attestation (§6) is not a renter-confidentiality control.** Attestation proves to Superheat and the renter that the node is running genuine, signed Superheat software (it defends against a **rogue/cloned node**, T9/T13). It does **not** protect renter data from the host, because the host owns the silicon the attested software runs on and can read the live memory of that software regardless of how it was measured at boot. For that reason attestation is **not** listed below as a confidentiality guarantee.

### 4.3 Two tiers — what each does and does NOT guarantee

This axis is the canonical **`confidentiality_class ∈ {standard, trusted_host}`** (`data-contracts.md`). It is a **security/hardware property, orthogonal to the reliability tier** (`reliability_tier ∈ {probation, spot, standard, premium}`, `data-contracts.md`): a node carries both, and **`trusted_host` must never appear in `reliability_tier_required`** — do not treat it as a reliability level. A job sets `confidentiality_required ∈ {standard, trusted_host}` (`data-contracts.md`) and the feasibility predicate (`data-contracts.md`) matches it against the node's `confidentiality_class`.

Renters **must** be told their confidentiality class before they accept a job, on every demand path (own API and via marketplace connectors where the field can be surfaced). It is a first-class attribute in the registry and on every metered line (`platform.md`).

| Property | **`standard` class** (home H1 + un-vetted nodes) | **`trusted_host` class** (vetted commercial / future TEE) |
|---|---|---|
| Tenant→host / →LAN / →thermal isolation | **Yes** (§3) | **Yes** (§3), VM-per-tenant baseline |
| Encryption at rest + in transit | **Yes** | **Yes** |
| No-persistence + wipe-on-job-end | **Yes** | **Yes** |
| Software attestation of node image/agent | **Yes** (§6) — anti-rogue-node, **not** a confidentiality control (§4.2) | **Yes** (§6) — same caveat |
| Confidentiality vs **determined host (live memory or physical access)** | **NO — explicitly not guaranteed** | **Strong**: vetted/contracted site, sealed/tamper-evident unit, restricted physical access; **hardware TEE where the GPU supports it** |
| Site vetting / operator agreement | None required | **Required** (commercial operator, contractual + physical-security controls) |
| Pricing | Baseline | **Premium**, tracked + metered separately |
| Suitable for | Non-sensitive training/inference, rendering, batch, open weights | Sensitive models/data, regulated workloads, customers who require it |

**Trusted-host vetting** = a commercial site under a Superheat operator agreement with: controlled physical access, tamper-evident enclosure, no host admin on the node, and (where hardware permits) an attestable GPU TEE. The tier is a *contractual + physical + (future) hardware* construct, not a software trick.

### 4.4 Future TEE path

We track confidential-computing-capable GPU hardware in `financial-and-roadmap.md`. As/if TEE-capable cards become viable in heater nodes, they slot into the trusted-host tier with **GPU attestation** added to the node attestation flow (§6), enabling genuine hardware-enforced confidentiality for that sub-pool. Until then, the standard-tier honesty statement (§4.1) stands.

### 4.5 Honest residual limitations (consolidated)

For diligence, the confidentiality residuals we do **not** fully close on a standard node:

- **Live memory during compute.** Data and model are in cleartext in RAM/VRAM while the job runs; a root host can read them (§4.1, §4.2). Encryption-at-rest and wipe-on-job-end do not cover this. No software control closes it on non-TEE hardware.
- **Attestation is not confidentiality.** It defends against rogue/cloned nodes, not against the host reading the attested software's memory (§4.2). Not a renter-confidentiality control.
- **VRAM residue across CUDA contexts.** Complete VRAM clearing on consumer 4090s is a partially-open problem (published residue research); scrub is best-effort, not guaranteed (§3.3).
- **Physical/bus attacks.** Cold-boot, DMA, and bus-probing on hardware the host owns cannot be cryptographically prevented on a standard node (§2.3).

All of the above are why sensitive work belongs on the **trusted-host tier** (`confidentiality_class = trusted_host`, §4.3) or off-network — not on a standard node.

---

## 5. Abuse prevention & ToS enforcement

Superheat must protect the homeowner (legal/reputational), the network (other tenants, our IPs), and the financing story (clean, abuse-free tape). KYC/AML, sanctions, lawful process, and per-jurisdiction illegal-content reporting are treated end-to-end in Part B §11; this section is the technical detection-and-response substrate they build on.

### 5.1 What we prohibit

- **Crypto mining** (economically nonsensical on rented GPU-hours and a common abuse vector).
- **Illegal content** — generation, storage, or distribution (CSAM and other illegal material).
- **Network abuse** — using the node as a **DDoS source**, port scanner, spam relay, or for unauthorized access to third parties.
- **Tampering** with the node, isolation, metering, or thermal safety.

### 5.2 Detection

- **Egress firewall signals (`connectivity-and-data.md`):** mining-pool destinations, abnormal connection fan-out (scanning), SMTP attempts, sustained high-rate outbound to many peers (DDoS), DNS abuse.
- **Telemetry signatures (`platform.md`):** mining has a distinctive sustained-100%-GPU + tiny-memory + low-I/O profile vs. legitimate training/inference; flag and review.
- **KYC + reputation on direct (own-API) renters.** **KYC does NOT transfer to marketplace-sourced renters** — on the marketplace path we do not know who the end renter is (the marketplace is the customer of record, and its KYC, if any, is not shared with us). So for marketplace demand, illegal-content/abuse control is **detection-and-kill, not KYC prevention**: marketplace-sourced tenants inherit our egress + telemetry controls and the §5.3 response ladder regardless of the marketplace's own policy, but there is no identity gate at intake. This is the honest abuse posture for the hybrid-demand model (principle 5, §1); the full KYC treatment is Part B §11.1.
- **Abuse-report intake** (homeowner, public, NCMEC/hotline, hosting-abuse contacts) wired to the on-call security queue.

### 5.3 Response ladder

1. **Throttle** suspicious egress/compute and raise an alert (auto).
2. **Suspend** the job; snapshot evidence to the audit log (§7) under legal hold if applicable.
3. **Kill + wipe** the sandbox (cryptographic erase, §4.2).
4. **Ban** the renter (across all demand paths) and, for illegal content, **preserve evidence** and follow the legal-escalation runbook (law-enforcement contact, NCMEC/per-region reporting where required — Part B §11.4).
5. **Protect the host:** because of outbound-only + egress isolation (§3.4), abuse never originated *from* the home LAN and never touched the homeowner's devices — a key point for homeowner indemnification and the ToS.

This ladder applies on **every demand path** including marketplace-sourced (no KYC there). What differs by region is the reporting destination and trigger (Part B §11.4).

### 5.4 Host indemnification posture

The homeowner agreement and renter ToS must state that (a) renter traffic is isolated from the home network, (b) Superheat owns abuse monitoring and response, and (c) the homeowner is not the operator of the renter's workload. The technical controls in §3.4/§3.5 are what make those contractual promises true — and they are the same controls that underwrite the insurance indemnity line (Part B §9.1) and shield the homeowner from lawful process aimed at a renter (Part B §11.3).

---

## 6. Node attestation & identity

**Principle: a node earns paid work only after proving it runs genuine, signed Superheat software and is the node it claims to be.** This protects renters (their job runs in a known-good sandbox), the operator (no rogue/cloned nodes draining the ledger), and the tape (metered hours come from attested nodes only).

### 6.1 Identity (coordinates with `node.md` + `connectivity-and-data.md`)

- Each node has a **unique cryptographic identity** bound at provisioning, stored in a **TPM / secure element where the board provides one**; otherwise a sealed software key with the strongest available binding (tracked as a hardware gap in `financial-and-roadmap.md`).
- The node identity key authenticates the **WireGuard overlay join** and the control-API session (mutual auth, cert-pinned) — see `connectivity-and-data.md`. **No shared fleet-wide secret** ever lives on a node (T14); each node's keys are least-privilege and node-scoped.

### 6.2 Attestation flow (before paid work)

1. **Secure boot + measured boot** of the immutable OS (`node.md`): firmware → bootloader → signed OS image → node agent, each measured.
2. Node agent reports **measurements + image version + agent signature** to the control plane over the authenticated channel.
3. Control plane verifies: signed image, expected measurements, agent integrity, identity-key validity, not-revoked. **Only attested nodes are marked eligible** by the scheduler/yield router (`platform.md`).
4. **Re-attestation** periodically and after every OTA; a node that fails attestation is **quarantined** (no paid work, heater keeps running — fail-safe) and flagged for triage.

### 6.3 OTA & supply-chain integrity

- **Signed images only**, A/B updates with auto-rollback (`node.md`); a node never accepts an unsigned or downgraded image (anti-rollback). Staged canary→cohort→fleet with auto-halt (`platform.md`). This blocks the "malicious OTA acceptance" path in T13.

### 6.4 Key management — rotation & revocation (coordinates with `connectivity-and-data.md`)

- **Rotation:** overlay and API keys rotate on a schedule and on OTA; per-job data-encryption keys (§4.2) are single-use — control-plane-derived, held only in node RAM for the job, dropped on job end.
- **Revocation:** a compromised/cloned/decommissioned node is **revoked centrally** — overlay membership pulled, identity blacklisted, scheduler eligibility removed — within minutes. Revocation lists are enforced at the overlay coordinator and control API.
- **Quarantine vs revoke:** quarantine = temporarily ineligible (failed attestation, anomaly) but recoverable; revoke = permanent (confirmed compromise/clone). Both are auditable events (§7).

---

## 7. Securitization-grade operational controls

The financing thesis (`../market-research/Superheat_Compute-Network_Debt-Engine_Onepager.md`) turns metered compute cash flows into a securitizable asset. Two of its named risks are **servicing concentration** ("investors must trust that Superheat can keep the nodes running") and **gain-on-sale / auditability**. Security and operability controls are therefore not just engineering hygiene — they are **diligence artifacts**.

### 7.1 Backup-servicer agreement + escrowed runbooks

So the fleet is operable even if Superheat fails (the explicit mitigant in the Debt-Engine one-pager §7):

- **Backup-servicer agreement** with a named third party able to assume fleet operation, contractually pre-arranged before issuance.
- **Escrowed runbooks + operational keys + software**: the control-plane operational procedures, signed-image build chain, attestation roots, and the *capability* to run OTA and metering are deposited with an escrow agent and released to the backup servicer on a defined trigger (Superheat default/insolvency). This makes the servicing moat *transferable*, which is what lets investors underwrite it.
- **Designed for transfer:** least-privilege RBAC, documented procedures, no undocumented tribal-knowledge dependencies, infrastructure-as-code for the control plane.
- **Fail-safe-to-heater is the ultimate backstop:** even with zero servicing, every unit reverts to an ordinary water heater (the "dual collateral" argument in the one-pager §5) — the downside asset is real.

### 7.2 Audit logging

- **Append-only, tamper-evident** audit log of: attestation results, OTA events, job lifecycle (start/stop/preempt/wipe), abuse events + responses, operator actions (RBAC + break-glass), key rotations/revocations, and per-node uptime/availability.
- **Operator actions are logged and reviewable** (insider threat T15); break-glass access requires after-the-fact review.
- Logs are retained for the financing/audit horizon and are queryable per node and per pool (`platform.md`). This append-only log is also the **disclosure substrate** for lawful process (Part B §11.3) and the evidence trail for breach notification (Part B §10.2).

### 7.3 How security/uptime evidence feeds the financing tape

- **Attested-hours-only metering, honestly qualified:** the metering/billing ledger (`platform.md`, the securitization tape) counts GPU-hours only from attested, non-quarantined nodes, and each metered fact is a **double-signed usage record** (`data-contracts.md` — node identity key signs the metered facts, control plane counter-signs after reconciliation). Metering integrity therefore rests on **both** the double-signed usage record **and** attestation. The "clean by construction" property is **strong only on the secure-element/TPM-equipped subset** of nodes, where the identity key is hardware-bound and the host cannot freely forge node signatures. On **software-only-bound nodes** (§6.1 hardware gap) a determined host who controls the box could in principle inflate reported GPU-hours by signing inflated usage records with a key it can reach. The **compensating control** is the cloud counter-signature plus **three-way reconciliation**: signed usage records vs. marketplace settlement statements vs. bank receipts (`data-contracts.md` reconciliation; `platform.md` demand routing) — fabricated hours that no buyer paid for fail to reconcile and are caught, and node anomaly detection (T13) flags the divergence. So the tape is **reconciled-clean**, not unconditionally clean-by-construction.
- **Reliability + security score per node** (uptime, attestation pass rate, abuse incidents, isolation health) seasons each node and sorts it into financing sub-pools (`system-design-overview.md`). High-reliability, abuse-free, attested nodes are exactly the seasoned pool underwriters want.
- **Abuse-free tape:** §5 enforcement keeps illegal/abusive use out of the revenue stream that gets packaged — a reputational and legal must for institutional buyers.
- **Confidentiality-class labeling on every line:** `confidentiality_class` = `standard` vs `trusted_host` (§4.3, `data-contracts.md`) is metered separately — orthogonal to the reliability tier — so `trusted_host` revenue (premium, contractual) can be pooled and underwritten distinctly.

---

## 8. Part A summary: the guarantees we make (and don't)

**To the homeowner/host — strong guarantees:**
- A renter's packets can never be addressed to your home network or devices (default-deny egress + no host networking + outbound-only) — §3.4.
- A renter can never reach or disable your hot-water service; local thermal safety overrides everything — §3.5, fail-safe-to-heater.
- You are not the operator of renter workloads; Superheat owns abuse monitoring and response, and abuse never originates from your LAN — §5, indemnity at Part B §9.1.

**To the renter — honest, tiered guarantees:**
- Strong isolation from the host OS, the host LAN, the thermal device, and other tenants — §3.
- Ephemeral, encrypted-at-rest, wiped-on-job-end data handling (per-job key from the control plane, held in node RAM only, dropped on job end — §4.2) and an attested, signed runtime — §6.
- **Attestation proves the node runs genuine Superheat software (anti-rogue-node); it is NOT a confidentiality guarantee against the host** — §4.2.
- **`standard` confidentiality class: NO confidentiality against a determined host — neither against live RAM/VRAM reads during your job, nor against physical access** — stated up front, every time — §4.1, §4.5. Use the **`trusted_host`** class if you need that.

**To Superheat / investors — operability guarantees:**
- Only attested, non-revoked nodes earn paid work and contribute to the tape — §6, §7.3.
- The fleet is operable by a backup servicer via escrowed runbooks/keys if Superheat fails; worst case, every unit is still a working water heater — §7.1.
- Every security-relevant event is auditable and feeds per-node/per-pool reliability scoring — §7.2/§7.3.

---

# Part B — Legal, Regulatory, Compliance & Insurance

## 9. Electrical, building code, permitting & inspection

The unit is a **fixed appliance on a branch circuit**, so it is squarely inside the National Electrical Code (NEC / NFPA 70 in the US, adopted with amendments by each Authority Having Jurisdiction — AHJ), the local building/plumbing code, and the per-install permit + inspection process.

### 9.1 The code facts that constrain the install (from `node.md`)

- **H1 (home):** GPU + host ≈ 600 W rides inside an existing **240 V / 30 A water-heater branch circuit**. The heating element dominates the draw; in most cases **no new service is required**, but the *combined* continuous load must still be evaluated against the existing circuit.
- **Commercial (8× 4090):** ~4 kW node ≈ ~17 A continuous on 240 V single-phase → **base config is one 240 V / 40 A circuit with margin**; larger sites use **208 V 3-phase**. Premium-redundant sites run two PSUs from **two separate circuits**.
- **NEC continuous-load rule (the trap).** A load on for ≥3 hours is a *continuous load*; the branch circuit and overcurrent device must be rated ≥**125%** of it. Superheat's compute is, by design, **near-continuous** (that is the whole utilization thesis). So sizing must use 125% of the *combined* heater+compute continuous draw — **not** the average, and **not** the heater alone. Undersizing here is the most common install-blocker and a fire-liability source.
- **Inrush / transients.** 4090 spikes (`node.md`) plus staggered soft-start are the installer's breaker-coordination concern; document the design point (450 W/GPU) used for sizing.

### 9.2 Permit + inspection workflow (per install)

| Step | Action | Owner | Gating artifact |
|---|---|---|---|
| 1 | Site electrical assessment (existing circuit, panel capacity, service type) | Installer (certified partner) | Site-assessment record in registry (`platform.md` `installer_id`) |
| 2 | **Pull electrical permit** (and plumbing/mechanical permit where the heater swap requires it) | **Installer** — licensed electrician/plumber of record | Issued permit number |
| 3 | Install per approved SKU spec + NEC/local amendments | Installer | As-built wiring/plumbing record |
| 4 | **Pass AHJ inspection** (electrical; plumbing/mechanical if applicable) | AHJ; installer schedules | **Signed inspection sign-off** |
| 5 | Energize compute (node leaves `installed` → `attested` → `active`, `platform.md` state machine) | Superheat (control plane) gated on artifact | Inspection sign-off recorded on the node record |

**Who pulls the permit:** the **licensed installer** (the partner electrician/plumber), never Superheat-the-platform and never the homeowner. This matches the partner-led-install model (`Superheat AI Data Center.md` Slides 6, 9) and keeps the licensed trade liable for code compliance. Installer certification + the per-install runbook is owned by `operations.md`.

**AHJ variance risk.** Code is adopted and amended *locally*; an AHJ may classify the unit as an unlisted appliance, a "data center" (triggering commercial occupancy rules), or refuse a residential GPU load outright. Mitigation: §10 listing removes most of this; maintain a **per-AHJ disposition log** so a refusal in one town is captured before it recurs. A pattern of refusals in a metro is a §15 region-entry blocker.

### 9.3 What blocks an install (electrical/building)

- No issued permit, or no passed inspection → **node cannot be energized for compute** (heater-only operation is allowed; compute is gated).
- Circuit cannot carry 125% of combined continuous load and customer declines a service upgrade → **decline the site** (or H1-only).
- AHJ classifies the install as requiring commercial occupancy/zoning the site doesn't have → **decline / escalate**.

---

## 10. Appliance safety & certification (UL/ETL, NSF, plumbing code)

The unit is sold and installed as a **water heater**. It must therefore meet the safety-certification expectations of a water heater **plus** those of the embedded electronics — as **one combined listed appliance**, not two stapled-together products.

### 10.1 Required listings/certifications

| Obligation | What it covers | Standard (illustrative — counsel/test-lab to confirm) | Owner | Gating artifact |
|---|---|---|---|---|
| **NRTL listing of the combined appliance** | Electrical safety of the heater+compute unit as one product | UL 174 / UL 1453 (electric water heaters) + UL 60950/62368-class for the IT load, evaluated together | Superheat (hardware) + NRTL (UL/ETL/CSA) | **UL/ETL Listing mark + report** for the SKU |
| **Potable-water contact** | Anything touching domestic hot water is health-safe | **NSF/ANSI 61** (drinking-water components); local plumbing code | Superheat (hardware) | NSF/ANSI 61 certificate for wetted parts |
| **Pressure/temperature relief** | T&P relief valve, tank pressure rating | ASME / plumbing-code T&P requirements | Superheat (hardware) | Listed T&P valve + tank pressure rating on the listing |
| **EMC / RF emissions** | The GPU host is IT equipment near a residence | FCC Part 15 (US), CE/UKCA (intl) | Superheat (hardware) | FCC ID / DoC; CE for EU |
| **Energy / efficiency labeling** | Water-heater efficiency claims | DOE/FTC EnergyGuide (US); ErP (EU) | Superheat (product/legal) | Compliant labeling |

The **separation of the two electrical domains** (`node.md` principle: heater control galvanically/logically separable from compute) is the design feature that makes a clean combined listing achievable — the heater's safety case does not depend on the GPU subsystem.

### 10.2 The fail-safe-to-heater requirement IS a certification claim

`node.md` and `system-design-overview.md` (principle 2) make **"any compute fault degrades to an ordinary working water heater"** a hardware safety property enforced by a **hardware safety interlock** in the thermal MCU (over-temp, no-flow, lost-heartbeat → cut GPU power; default-safe to heater-only). This is the same interlock that sits on the tenant side of the isolation boundary in Part A §3.5.

For certification and insurance this stops being just an engineering invariant and becomes a **claim we must be able to substantiate**:
- The interlock must be **independent of the host OS and the cloud** (so the safety case survives a software/comms failure).
- It must be **documented in the listing's safety analysis** and reproducible in test-lab evaluation.
- It is the technical basis for the homeowner promise "you will never lose hot water" and for the insurer's product-liability underwriting (§11... see §11 below / §13 insurance).
- **Regression of this property is a Critical risk** (`financial-and-roadmap.md` R9) and a **standing release gate** — never ship an OTA that can defeat the interlock.

### 10.3 The risk narrative we must own: "GPUs in a water tank"

The single most scrutinized phrase for inspectors, insurers, and rating agencies. The honest, defensible position:
- The GPUs are **coupled to the water via the heat path, not immersed in potable water** (the coldplate/heat-exchanger loop is separate; wetted parts are NSF-61). We must be able to state and certify the separation precisely.
- Combined electrical + water in a residence raises **fire and water-damage** exposure; the listing + the fail-safe interlock + GFCI/leak-detection design are the answers.
- Marketing must **not** outrun the listing. Until a SKU carries its mark, it cannot be sold as a listed appliance in jurisdictions that require listing.

### 10.4 Confidentiality honesty carries a legal edge

Part A §4.1 is explicit: on a **`standard` node we cannot guarantee confidentiality against a determined host.** Legally this means we must **not** represent standard nodes as offering confidentiality we cannot deliver (a misrepresentation/UDAP risk), and regulated renter data (HIPAA, financial, etc.) belongs on the **`trusted_host`** class or off-network. The per-job-key lifecycle and wipe-on-job-end (Part A §4.2) are the substantiation for our at-rest/deletion claims.

### 10.5 What blocks an install / a securitization (appliance safety)

- **No NRTL listing for the SKU** → most AHJs will not pass inspection (§9) → **install-blocking**; and underwriters will not pool an unlisted appliance → **securitization-blocking**.
- No NSF-61 on wetted parts → plumbing-inspection failure.
- A listing that does **not** cover the as-shipped configuration (e.g., a year-5 GPU swap to a card outside the listed envelope) → re-evaluation required before that config ships (ties to `node.md` modular swap and §16 below).

---

## 11. KYC/AML, lawful process & illegal-content obligations

This section operationalizes the **marketplace-KYC gap** and the abuse/lawful-process controls from Part A §5.

### 11.1 KYC — and the gap that does not transfer

| Demand path | Who the customer is | KYC posture | Control |
|---|---|---|---|
| **Direct / own-API renters** | Known to Superheat | **KYC at intake** + sanctions screening + reputation | Identity gate before paid work (Part A §5.2) |
| **Marketplace-sourced renters** (vast.ai, RunPod, …) | **Unknown** — the marketplace is the customer of record; its KYC (if any) is **not shared** | **KYC does NOT transfer.** No identity gate. **Detection-and-kill only.** | Egress + telemetry detection + the Part A §5.3 response ladder |

This is the honest abuse posture for the hybrid-demand model. On the marketplace path we control **behavior**, not **identity**: isolation, egress firewall, abuse detection, rapid kill + evidence preservation — applied regardless of the marketplace's own policy.

### 11.2 Sanctions screening (OFAC and equivalents)

- **Direct renters and all homeowners/installers (payees)** are screened against OFAC SDN and equivalent lists at onboarding and on a refresh cadence. We pay homeowners and installers — **paying a sanctioned party is itself a violation**, so the **payout rail** (`financial-and-roadmap.md` line 7, Stripe Connect/ACH) is a screening checkpoint, not just the renter intake.
- Region/IP-level controls block embargoed jurisdictions on the renter side.

### 11.3 Lawful process: preservation, intercept, disclosure

- **Preservation & legal hold:** the abuse response ladder (Part A §5.3) already snapshots evidence under legal hold and preserves it for law enforcement. The **append-only audit log** (Part A §7.2) is the disclosure substrate.
- **Lawful-intercept / disclosure hooks:** Superheat (as operator) — not the homeowner — responds to valid legal process. Because traffic is mediated through the overlay and the control plane (`connectivity-and-data.md`), Superheat is the technically-correct point of compliance. Obligations (what process is valid, what can be intercepted vs only preserved, notice requirements) **differ by jurisdiction** — owned by legal, with a documented LE-response runbook.
- **The homeowner is shielded:** outbound-only + egress isolation (Part A §3.4) means abuse never originated from the home LAN — load-bearing for keeping the homeowner out of legal process aimed at a renter.

### 11.4 Illegal content (CSAM and others) — obligations DIFFER by jurisdiction

| Obligation | US | EU / other (illustrative) | Owner |
|---|---|---|---|
| **CSAM** detection/handling on discovery | Report to **NCMEC**; preserve; do not re-distribute | National hotlines; EU rules evolving | Superheat security + legal |
| **Mandatory reporting trigger** | On actual knowledge (US providers) | Varies; some require proactive measures | Legal (per-region) |
| **Other illegal content** | Varies by content + state | Varies widely | Legal (per-region) |

The **Part A §5.3 response ladder** (throttle → suspend → kill+wipe → ban + preserve + report) applies on **every demand path** including marketplace-sourced (no KYC there). **What differs by region is the reporting destination and trigger**, which is a per-region legal artifact in the §15 checklist. NCMEC reporting and the LE escalation runbook are owned jointly by security (Part A §5) and legal.

### 11.5 What blocks an install / securitization (KYC/lawful)

- No sanctions screening on payout rails → **securitization-blocking** (AML/sanctions failure taints the cash flows and the institutional buyers).
- No documented illegal-content reporting runbook for a region → **region-entry-blocking** (§15).

---

## 12. Data privacy & compliance

A Superheat node carries **two** distinct categories of regulated personal data, and both live on **someone else's premises**. This section ties directly to the isolation and key-custody controls in Part A.

### 12.1 The two data subjects

1. **Renter data** — code, datasets, model weights transiting and residing (ephemerally) on a third-party residential node. May itself contain personal data of the renter's own users. Superheat is typically a **processor/service provider** for renter data; the renter is controller.
2. **Homeowner occupancy/behavior data — the non-obvious one.** The thermal forecaster and tank-as-battery model (`platform.md` thermal coordinator / utilization engine) **inevitably learn hot-water draw and heat-demand patterns**, which are a high-fidelity proxy for **presence, occupancy, sleep/wake, shower times, and away periods**. This is personal data about the homeowner (and a physical-security signal). Superheat is **controller** for it.

> **Flagged gap to own:** the occupancy-inference property of the thermal model is a privacy exposure that the security and control-plane docs touch only implicitly. It must be governed: minimize retention of fine-grained draw data, aggregate/derive only what scheduling needs, disclose it in the host privacy notice, and treat the raw draw timeseries as sensitive in access control and the breach plan.

### 12.2 Obligations → owner → gating artifact

| Obligation | Applies to | Owner | Gating artifact |
|---|---|---|---|
| **Lawful basis + notice** (homeowner occupancy data) | Homeowner | Superheat (legal/product) | Host privacy notice + consent in host agreement |
| **Renter DPA / processor terms** | Renter data | Superheat (legal) + `platform.md` (renter API) | Signed DPA / ToS data-processing terms |
| **GDPR** (EU/UK) — lawful basis, DSAR, DPA, possibly DPO/representative | Both, if EU data/subjects | Superheat (legal) | Records of processing; SCCs for transfers |
| **CCPA/CPRA + US state privacy laws** | CA + ~20 states | Superheat (legal) | Consumer-rights + opt-out mechanism; service-provider terms |
| **Data residency** | Renter data on a node physically *in* a jurisdiction | Superheat (control plane placement) | Region-aware placement policy (§12.3) |
| **Breach notification** | Both | Superheat (security) | Incident-response plan with jurisdiction-specific clocks |
| **Retention & deletion** | Both | Superheat (security/data) | Retention schedule; wipe-on-job-end evidence (Part A §4.2) |

### 12.3 Data residency — a physical-placement problem, not just a policy

Because the GPU is **physically located** in a specific jurisdiction, renter data **resides** there the moment a job lands on that node. This makes residency a **scheduler constraint**:
- A job with a residency requirement (e.g., "EU-only," "US-only") must be a **placement predicate** in the yield router / scheduler (`platform.md` feasibility predicate), matching `node.region` against the job's `data_residency_required`.
- Cross-border compute is a cross-border data transfer — surface it; do not silently route an EU renter's job to a US basement.

### 12.4 What blocks an install / securitization (privacy)

- No host privacy notice/consent covering occupancy data → **install-blocking** in privacy-law jurisdictions.
- No breach-response plan / DPA → **securitization-blocking** (institutional buyers diligence this).

---

## 13. Insurance & liability

The combined appliance creates **product-liability** exposure (fire, electrical, water damage) on **third parties' premises**, layered on top of a multi-tenant compute service. Insurance is both a cost line (`financial-and-roadmap.md` line 10, ~$110/unit-yr `[A]`) and a **securitization requirement** (a financed asset that destroys the host's home is a defaulted, litigated asset).

### 13.1 Who is liable for what, and whose policy responds

| Risk event | Primary responsible party | Policy that should respond | Gating artifact |
|---|---|---|---|
| Unit causes **fire / electrical fault** | Superheat (manufacturer of the listed appliance) | **Superheat product-liability / general-liability** | Active product-liability policy ≥ required limits per SKU/region |
| Unit causes **water damage / leak** | Superheat (product) + installer (workmanship) | Superheat product policy; installer's policy for install defects | Both policies on file |
| **Installation defect** (bad wiring, bad plumbing) | **Installer** (licensed trade of record) | **Installer's liability + workmanship** insurance | Certificate of insurance (COI) on file before any install |
| Damage to homeowner's other property | Superheat product policy (if caused by unit) | Superheat; homeowner's HO policy secondary | — |
| **Hardware loss/theft/damage** to the GPU asset | Superheat (asset owner / SPV) | **Equipment / property insurance** on the fleet (the collateral) | Per-asset coverage naming the SPV/lender as loss payee |
| **Renter workload causes harm** (illegal use, network abuse) | Renter; Superheat as operator | Superheat tech-E&O / cyber; abuse controls (Part A §5) | ToS + abuse runbook |
| **Renter data breach** on a node | Superheat (operator) | **Cyber / tech-E&O** | Breach-response plan (§12) |

**Homeowner policy vs Superheat policy — the line we must hold.** The homeowner's standard HO policy will likely **exclude** a commercial/income-producing appliance and the renter's commercial activity. We must **not** rely on the homeowner's policy for product or operational risk. Superheat carries the product, operational (cyber/E&O), and equipment coverage; the **host agreement** must (a) disclose the appliance, (b) confirm Superheat indemnifies the host for product/operational harm, and (c) confirm the host is **not the operator** of renter workloads (mirroring Part A §5.4 — the technical egress isolation in Part A §3.4 is what makes the indemnity *true*).

### 13.2 Warranty

- **Appliance/heating warranty** to the customer (the heater must keep heating) — owned by Superheat product; serviced via the installer network. Fail-safe-to-heater (§10.2) bounds the worst case.
- **Compute-availability** is **not** a consumer warranty to the homeowner — homeowner compute income is *upside*, explicitly variable (consistent with `Superheat AI Data Center.md` Slide 8 "can offset… may cover"). Over-promising compute income is a **consumer-protection / UDAP** risk (§14) — marketing must frame it as variable.
- **Renter SLA** is a separate, tiered commercial promise (reliability tier, `confidentiality_class`) — owned by `platform.md` (renter API) / Part A §4.3, not the homeowner warranty.

### 13.3 What blocks an install / securitization (insurance)

- No installer COI on file → **no install** (the partner-network onboarding gate).
- No Superheat product-liability + equipment coverage at required limits → **securitization-blocking** (lenders require the collateral and the liability be insured; the SPV/lender is named loss payee).

---

## 14. Consumer financing regulation

The unit is **financed at the point of sale** (`Superheat AI Data Center.md` Slide 7: "as easy to finance as solar panels," ~$88–132/mo, financed over 10 years). Consumer lending is heavily regulated and is a distinct legal world from everything above.

### 14.1 Obligations

| Obligation | What it requires | Owner | Gating artifact |
|---|---|---|---|
| **TILA / Regulation Z** (US) | Truth-in-Lending disclosures: APR, finance charge, total of payments, right of rescission (for home-secured) | Lending partner (preferred) or Superheat-as-lender | Compliant loan disclosures at POS |
| **State lender / sales-finance licensing** | A license to originate or facilitate consumer credit — **state by state** | Lender of record | Licenses in each state of operation |
| **ECOA / fair-lending** | No prohibited-basis discrimination; adverse-action notices | Lender | Underwriting + adverse-action process |
| **UDAP / UDAAP** (FTC + CFPB + state AGs) | No deceptive marketing — esp. **compute-income claims** | Superheat (marketing) + lender | Substantiated, variable-income disclosures (§13.2) |
| **Right to repair / lease-vs-loan characterization** | Whether the deal is a loan, lease, or service contract changes which rules apply | Legal + structuring | Documented product characterization |

### 14.2 Strategic recommendation: do not become the lender

Originating consumer credit means acquiring and maintaining **state-by-state lending licenses** — a large, slow, ongoing compliance burden. **Recommendation: use a licensed point-of-sale lending partner** (the "solar finance" pattern) so the licensing, TILA disclosures, and fair-lending obligations sit with a regulated lender, and Superheat is the merchant/originator of the *asset*, not the *credit*. This is also cleaner for the securitization: the **compute-revenue securitization** (the SPV/ABS in `financial-and-roadmap.md`) is distinct from the **consumer loan** on the appliance, and they should be kept legally separate to avoid the consumer-lending rules contaminating the compute-ABS (and vice-versa).

### 14.3 The income-claim line (ties §13.2 + §12.4)

"Compute income helps repay the loan" is a **financing + advertising** statement. If it functions as an inducement to borrow, mis-stating it is both a UDAP and a lending violation. Keep all income figures explicitly **variable, illustrative, not guaranteed** — consistent across the deck, the loan disclosures, and the host agreement.

### 14.4 What blocks an install / securitization (financing)

- POS financing offered in a state without the lender's license → **install-blocking in that state** (can still sell cash/unfinanced).
- Consumer-loan terms entangled with the compute-ABS structure → **securitization risk** — keep separate.

---

## 15. Tax

Three taxable relationships per unit: the **homeowner's compute income**, **Superheat's asset** (depreciation/credits), and **sales/use tax** on the compute service — across borders.

| Tax obligation | Who | Treatment / requirement | Owner | Gating artifact |
|---|---|---|---|---|
| **Homeowner compute payouts = taxable income** | Homeowner (payee) | Information reporting — **1099-class** (e.g., 1099-NEC/MISC, or 1099-K via the payout processor) when thresholds are met; collect **W-9/W-8** at onboarding | Superheat (finance/payout) | W-9/W-8 on file; year-end 1099 issuance |
| **Backup withholding** | Homeowner | Required if TIN missing/invalid | Payout processor / Superheat | TIN matching |
| **Depreciation / cost recovery** on GPU + heater asset | Superheat / SPV (asset owner) | MACRS / bonus depreciation on the fleet | Finance/tax | Fixed-asset register tied to registry |
| **Energy/Investment Tax Credit eligibility** | Superheat / SPV | **Uncertain** — a GPU-in-a-heater is not obviously ITC-eligible HVAC/energy property; do **not** assume the credit | Tax counsel | Written eligibility opinion before any ITC is modeled into the tape |
| **Sales/use tax on compute services** | Renter (collected by Superheat) | Taxability of IaaS/compute **varies by US state** and country (VAT/GST abroad); economic-nexus thresholds | Superheat (finance) | Per-state taxability matrix + collection in billing |
| **Sales/use tax on the appliance sale** | Customer | Taxable tangible property in most states | Installer/Superheat | Collected at POS |
| **Cross-border** | All | Permanent-establishment risk from physical nodes abroad; VAT/GST registration; withholding on cross-border payouts | Tax counsel | Per-country tax opinion (§16) |

> **Two tax items that touch the tape directly:** (1) **Do not bank the ITC** without a written opinion — over-counting credits inflates the underwritten asset yield. (2) **Sales tax on compute** must be collected correctly per state or it becomes a contingent liability against the SPV's cash flows. Both feed `financial-and-roadmap.md`.

**Cross-border PE warning.** Placing income-producing GPU nodes in another country can create a **taxable permanent establishment** and VAT/GST obligations there. International expansion is **not** a marketing decision — it is gated on a per-country tax opinion (§16).

---

## 16. Jurisdiction & multi-region matrix

Every obligation above varies by jurisdiction. The fleet's defining feature — **thousands of units across many AHJs, states, and countries** — multiplies compliance surface. The control is **per-region gating**: codify obligations per region, and do not enter a region until its blocking items have owners and artifacts.

### 16.1 How the obligations vary (the axes)

| Domain | US state-by-state variation | International variation |
|---|---|---|
| Electrical/building code (§9) | NEC adoption year + local AHJ amendments; permit/inspection process | Different code regimes entirely (IEC, CE, local) |
| Appliance listing (§10) | NRTL listing generally national; some local quirks | UL not recognized abroad — need **CE/UKCA/CSA/country marks** |
| Insurance (§13) | Limits/requirements vary; some states' regs differ | Different liability regimes; local insurer requirements |
| Privacy (§12) | ~20 state privacy laws (CCPA/CPRA + others) | **GDPR/UK GDPR**, data-localization laws |
| KYC/illegal-content (§11) | Mostly federal (OFAC, NCMEC) | Differing reporting triggers + intercept law |
| Consumer finance (§14) | **Heavily state-by-state** licensing | Different consumer-credit regimes |
| Tax (§15) | Compute-tax + nexus vary by state | VAT/GST, PE, withholding |

### 16.2 The implication

- **US expansion** is gated primarily by **(a)** the lender's state lending license (§14), **(b)** AHJ/permit feasibility (§9), and **(c)** the per-state compute sales-tax + privacy-law posture.
- **International expansion** additionally requires a **new appliance certification** (UL is not recognized — CE/UKCA/country marks), GDPR-class privacy compliance, and a **tax/PE opinion** — a far higher bar. Treat "global" as a deliberate, separately-gated program, not an extrapolation of the US rollout.

### 16.3 Per-region entry checklist (compliance gating)

A region (US state or country) is **green-lit for deployment** only when every blocking item below has a named owner and a satisfied artifact. **Until then, do not install or rent in that region.**

| # | Gate item | Domain | Blocking? | Owner | Artifact required |
|---|---|---|---|---|---|
| R1 | SKU carries a recognized appliance listing valid in the region | §10 | **Install-block** | Hardware | UL/ETL (US) or CE/UKCA/CSA (intl) for the SKU |
| R2 | Permit + inspection pathway confirmed with sample AHJ(s) | §9 | **Install-block** | Field-ops / installer | AHJ disposition + a passed sample install |
| R3 | Certified installer(s) with valid trade license + COI | §9, §13 | **Install-block** | Field-ops | Installer license + insurance certificate |
| R4 | Superheat product-liability + equipment coverage extends to region | §13 | **Install + securitization** | Finance/legal | Policy endorsement |
| R5 | Host agreement + privacy notice valid under region's privacy law | §12 | **Install-block** | Legal | Reviewed host agreement |
| R6 | Renter data-residency placement rule configured if region restricts | §12 | **Rent-block** | Control plane | Scheduler residency predicate |
| R7 | Illegal-content reporting runbook + reporting destination for region | §11 | **Rent-block** | Security/legal | Per-region LE/CSAM runbook |
| R8 | Sanctions/embargo posture confirmed (region not embargoed) | §11 | **Both** | Legal/compliance | Screening rules live |
| R9 | POS lender licensed in the region (if financing offered) | §14 | **Finance-block** (cash sales OK) | Lending partner | State/country lending license |
| R10 | Compute sales-tax / VAT-GST taxability + collection configured | §15 | **Rent-block** | Finance | Per-region tax matrix + billing config |
| R11 | Tax/PE opinion (international only) | §15 | **Install-block (intl)** | Tax counsel | Written opinion |
| R12 | Homeowner payout tax reporting (W-9/W-8, 1099 path) configured | §15 | **Securitization** | Finance/payout | Onboarding + reporting flow |

> **Rule:** a region's deployment status is the **AND** of its blocking gates. Track this as a per-region row in the control-plane registry alongside `node.region`, so a node can be created only in a green-lit region. This is the legal analog of the financing-tag gating in `financial-and-roadmap.md`.

---

## 17. RACI — ownership of every obligation

**R** = Responsible (does the work), **A** = Accountable (owns the outcome), **C** = Consulted, **I** = Informed.
Roles: **HW** (hardware/product), **FO** (field-ops / installer network), **SEC** (security/trust), **CP** (control-plane eng), **FIN** (finance/structuring), **LEG** (legal/compliance counsel), **INST** (licensed installer partner), **LEND** (POS lending partner).

| Obligation | HW | FO | SEC | CP | FIN | LEG | INST | LEND |
|---|---|---|---|---|---|---|---|---|
| NEC sizing / continuous-load (§9.1) | C | A | | I | | C | **R** | |
| Permit pull + inspection (§9.2) | I | **A** | | I | | C | **R** | |
| Appliance listing UL/NSF (§10) | **R/A** | I | C | | C | C | | |
| Fail-safe-to-heater certification claim (§10.2) | **R/A** | | C | C | | C | | |
| Product-liability + equipment insurance (§13) | C | I | C | | **A** | **R** | | |
| Installer COI / workmanship liability (§13.1) | | **A** | | | C | C | **R** | |
| Renter-data privacy / DPA (§12) | | | **R** | C | | **A** | | |
| Homeowner occupancy-data governance (§12.1) | C | I | **R** | **R** | | **A** | | |
| Data-residency placement (§12.3) | | | C | **R/A** | | C | | |
| Breach notification (§12.2) | | | **R** | C | I | **A** | | |
| KYC on direct renters (§11.1) | | | **R** | C | C | **A** | | |
| Sanctions screening (incl. payouts) (§11.2) | | | C | C | **R** | **A** | | I |
| Lawful process / intercept / preservation (§11.3) | | | **R** | C | | **A** | | |
| Illegal-content / CSAM reporting (§11.4) | | | **R** | | | **A** | | |
| TILA/Reg Z + state lending licensing (§14) | | I | | | C | **A** | | **R** |
| UDAP / income-claim accuracy (§13.2, §14.3) | | C | | | C | **A** | | C |
| Homeowner 1099 / W-9 / withholding (§15) | | | | C | **R/A** | C | | |
| Depreciation / ITC opinion (§15) | C | | | | **R** | **A** | | |
| Sales/use tax on compute + appliance (§15) | | C | | C | **R/A** | C | C | |
| Cross-border tax / PE (§15) | | | | | C | **A** (w/ tax counsel) | | |
| Per-region entry gating (§16.3) | C | C | C | **R** (registry) | C | **A** | | C |
| Tenant isolation / egress firewall (Part A §3) | C | | **R/A** | C | | I | | |
| Node attestation & key custody (Part A §4.2, §6) | C | | **R/A** | **R** | | I | | |
| Backup-servicer / escrowed runbooks (Part A §7.1) | | | C | **R** | **A** | C | | |

---

## 18. What blocks an install vs renting vs a securitization (summary)

**Blocks an INSTALL (the unit cannot be energized for compute on a customer's premises):**
1. No issued permit / failed AHJ inspection (§9).
2. SKU not listed (UL/ETL; CE/UKCA abroad) where listing is required (§10).
3. No certified installer with license + COI (§9, §13).
4. No host agreement + privacy notice valid in the region (§12).
5. Region not green-lit on its blocking gates (§16.3 R1–R3, R5, R11).
6. Node fails attestation / isolation health check (Part A §3.1, §6) → compute gated, heater-only.

**Blocks RENTING on a node (compute cannot be sold/routed there):**
7. No data-residency rule where the region requires it (§12.3).
8. No illegal-content reporting runbook for the region (§11.4).
9. Compute sales-tax/VAT not configured (§15).
10. Region embargoed / sanctions posture unresolved (§11.2).

**Blocks a SECURITIZATION (the unit/revenue cannot enter a financed pool):**
11. No product-liability + equipment insurance naming the SPV/lender (§13.3).
12. Sanctions/AML failure on the cash flows (§11.2).
13. Unsubstantiated ITC or mis-collected sales tax inflating/contaminating the tape (§15).
14. Consumer-loan rules entangled with the compute-ABS (§14.2).
15. No breach-response / DPA / lawful-process runbook (institutional diligence, §11, §12).
16. Listing that doesn't cover the as-shipped config, incl. a year-5 GPU swap outside the listed envelope (§10.5, `node.md`).
17. No backup-servicer / escrowed runbooks (servicing-concentration risk, Part A §7.1); abuse-tainted or unreconciled tape (Part A §7.3).

---

## 19. Open questions, risks & security questions (track in `financial-and-roadmap.md`)

### 19.1 Open security questions (Part A)

- **Stable consumer-GPU VFIO passthrough** in microVMs/Kata across the heater node board variants — gates VM-per-tenant on commercial nodes (§3.2).
- **Hardware root of trust coverage:** not all candidate boards ship a TPM/secure element; quantify the fraction lacking strong key binding and the residual identity risk (§6.1).
- **TEE-capable GPU roadmap** for the trusted-host tier — when does hardware-enforced confidentiality become economically viable in a heater node (§4.4)?
- **Marketplace tier-disclosure:** ensuring third-party marketplace listings can surface the standard-vs-trusted-host tier so renters always know their tier (§4.3) regardless of demand path.
- **Side-channel hardening** on multi-tenant commercial nodes beyond one-tenant-per-GPU (§3.3/§3.6).
- **Escrow trigger + backup-servicer drill:** rehearse an actual handoff so the §7.1 control is proven, not just papered.

### 19.2 Legal/regulatory risk register (Part B)

| # | Risk / open question | Impact | Owner | Resolving artifact |
|---|---|---|---|---|
| L1 | **NRTL listing of a novel combined heater+GPU appliance** — timeline, cost, and whether existing standards even fit; gates §9 inspections and §10 sales. | Critical (install + sale) | HW + LEG | Engaged test lab + listing roadmap before Phase-A scale |
| L2 | **AHJ classification risk** — some AHJs may treat the unit as a data center / unlisted appliance and refuse residential installs. | High | FO + LEG | Per-AHJ disposition log; pattern triggers §16.3 review |
| L3 | **State consumer-lending licensing** — breadth and time to license; argues for the partner-lender path (§14.2). | High | FIN + LEND | Lender's license footprint mapped to rollout states |
| L4 | **Occupancy-data privacy** — the thermal model's presence-inference is an unowned exposure today (§12.1). | Med–High | SEC + CP + LEG | Retention + disclosure policy; access controls on draw timeseries |
| L5 | **ITC / depreciation eligibility** of a GPU-in-a-heater — do not bank credits without an opinion (§15). | High (tape) | FIN + tax counsel | Written tax opinion before modeling into the tape |
| L6 | **Compute sales-tax matrix** across states + VAT/GST abroad — taxability of IaaS varies widely (§15). | Med | FIN | Per-region taxability matrix wired into billing |
| L7 | **International certification + PE** — UL not recognized abroad; physical nodes create PE/VAT exposure (§16.2). | High (intl) | LEG + tax | CE/UKCA roadmap + per-country opinions; gate intl as a separate program |
| L8 | **Marketplace illegal-content exposure** — no KYC on marketplace renters; reliance on detection-and-kill, with per-jurisdiction reporting differences (§11). | High | SEC + LEG | Per-region reporting runbook; tested abuse response (Part A §5) |
| L9 | **Listing vs year-5 GPU swap** — a refreshed card may fall outside the listed configuration (§10.5). | Med | HW + LEG | Listing envelope that covers the swap card, or re-evaluation in the refresh program |

---

*Cross-referenced docs:* `node.md` (immutable OS, node agent, attestation root, electrical/3-phase, UL, fail-safe interlock, year-5 GPU swap), `connectivity-and-data.md` (outbound-only overlay, egress firewall, key management, ephemeral storage, wipe-on-job-end, regional caches), `platform.md` (registry, region tagging, scheduler/yield router + feasibility predicate, audit ledger, metering/billing, demand routing, renter API/SLA), `data-contracts.md` (confidentiality_class, reliability_tier, double-signed usage records, reconciliation), `financial-and-roadmap.md` (phasing, residual + legal/regulatory risk register L1–L9, insurance OPEX, payout/sanctions rails, sales-tax/ITC into the tape), `operations.md` (installer certification + per-install permit/inspection runbook, incident response), `system-design-overview.md` (design principles, pooling/seasoning), `../market-research/Superheat_Compute-Network_Debt-Engine_Onepager.md` (securitization thesis, servicing concentration, dual collateral).

*This document is engineering/ops guidance — including on regulatory obligations and ownership — and is **not legal advice.** Every obligation marked as owned by **LEG/FIN/tax counsel** must be confirmed with qualified counsel in each jurisdiction before deployment, financing, or marketing.*
