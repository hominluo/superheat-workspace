# Superheat — Connectivity & Data I/O

**Status:** Draft v0.1 · **Date:** June 2026 · **Audience:** Engineering
**Scope:** This document merges the two tightly-coupled layers that sit between a renter
and a basement GPU: (A) the **connectivity / overlay-network layer** that solves NAT and
makes an unreachable residential node addressable and rentable; and (B) the **data I/O
layer & renter UX** that hides the consumer-bandwidth bottleneck and makes renting a
basement GPU feel like renting a data-center GPU. It expands §5 ("Connectivity layer —
solving NAT") and §9 ("Data I/O") of `system-design-overview.md` into one concrete,
buildable design.

The two parts are deliberately co-located because they share one economic constraint: the
**relay-bandwidth opex** of the overlay (Part A §A6) and the **bulk-off-relay / cache
strategy** of the data layer (Part B) are two halves of the same problem. The whole point of
the cache hierarchy and checkpoint dedup is to keep gigabytes *off* the paid relays, so the
relay carries only tiny interactive control traffic.

**Siblings:** `node.md` (the agent that owns the tunnel, the immutable OS, the egress
firewall, the NVMe hardware floor), `platform.md` (control plane / scheduler / reliability
scoring / yield router / renter API — consumes link + bandwidth telemetry and drives the
coordination API), `security-and-compliance.md` (segmentation, isolation, attestation,
encryption, wipe-on-end, trusted-host tier), `data-contracts.md` (the schemas exchanged),
`system-design-overview.md` (master).

---

## Design principles (shared across both parts)

The five overview principles shape this layer end to end. They apply equally to the overlay
and to data movement:

1. **Outbound-only.** The node opens connections; nothing is ever opened *to* the home
   router. This shapes both NAT traversal (the node dials the coordinator) and data movement
   (the node dials the PoP to pull and push). (Overview principle 1.)
2. **Fail-safe to heater.** Neither connectivity nor data I/O may ever block the heater or
   the watchdog. If connectivity is lost, the node degrades to an ordinary water heater with
   compute disabled — the customer always has hot water. Data I/O is the lowest-priority
   local consumer. (Overview principle 2.)
3. **Thermally coupled.** I/O budgets and checkpoint cadence are sized against the thermal
   window length from the utilization engine (in `platform.md`).
4. **Measure-then-price.** Real per-node link quality and bandwidth are first-class metrics
   feeding the reliability score in `platform.md` — not the ISP's advertised rate.
5. **Hybrid demand.** The same overlay and the same data layer serve marketplace jobs
   (vast.ai, RunPod) and our own backlog identically.

From these, the connectivity layer derives four hard requirements:

1. **Outbound-only** (above).
2. **Stable identity independent of IP.** A node is addressed by a cryptographic identity /
   overlay IP, not by where it happens to be.
3. **Direct-path-preferred, relay-capable.** Maximize peer-to-peer (cheap), degrade
   gracefully to relays (expensive but always works) — because CGNAT will defeat
   hole-punching on a meaningful slice of the fleet.
4. **Hard segmentation.** Control-plane traffic, renter-session traffic, and the homeowner
   LAN are three mutually-isolated domains.

And the data layer derives three:

1. **Never pull big artifacts over a home uplink if a nearby cache has them.** (Part B §B1)
2. **Never lose hours of work to a preemption.** Checkpoints are mandatory and must land on
   *local NVMe first*, uploaded asynchronously. (Part B §B2)
3. **Place data-heavy work where the bandwidth is.** (Part B §B3)

---

# Part A — Connectivity / Overlay Network

## A1. The problem, restated as constraints

Every Superheat node lives behind ordinary residential internet. That means, concretely:

- **No inbound reachability.** The home router does NAT; an unknown but large fraction of
  nodes are behind **CGNAT** (carrier-grade NAT) where even UPnP/port-forward is impossible
  because the public IPv4 is shared across hundreds of subscribers.
- **Dynamic IPs.** A node's apparent public address changes on lease renewal / reboot; we can
  never address a node by IP.
- **Asymmetric links.** Typical consumer plans are 200–500 Mbps down / 10–35 Mbps up.
  **Upload is the scarce resource** and it is shared with the homeowner.
- **The home LAN is not ours.** The homeowner's TVs, laptops, and cameras share the L2
  segment. We must never become a pivot into it (`security-and-compliance.md`, three-way
  isolation).

The whole layer is **one WireGuard-based overlay mesh** that satisfies the four connectivity
requirements above. WireGuard is the right primitive: kernel-fast, UDP-only (NAT-friendly),
tiny attack surface, key-pair identity built in, and roams across IP changes without tearing
down sessions.

---

## A2. Topology

```
                         ┌──────────────────────────────────────────────┐
                         │              CONTROL PLANE (cloud)            │
                         │  registry · scheduler · reliability scoring   │  platform.md
                         │  COORDINATION SERVER (overlay control)        │
                         └───────┬───────────────────────────┬──────────┘
                                 │ overlay (control subnet)   │
              ┌──────────────────┘                            └──────────────────┐
              │                                                                   │
   ┌──────────▼──────────┐   regional, per-geo                       ┌────────────▼─────────┐
   │  RELAY / DERP POP    │   (us-east, us-west, eu, ...)            │  REGIONAL DATA CACHE  │ Part B
   │  (TURN-like fallback)│◄──── bulk pulls go here, NOT over ─────► │  (model/img/dataset)  │
   └─────▲────────▲───────┘        the home uplink                   └───────────────────────┘
         │        │
   relay │        │ relay (only when direct fails)
         │        │
 ┌───────┴──┐  ┌──┴───────┐        direct p2p (hole-punched, preferred)
 │  NODE A  │  │  NODE B  │ ◄───────────────────────────────────────────────► RENTER / OPERATOR
 │ heater + │  │ heater + │                                                    (also overlay peers)
 │  4090    │  │  4090    │
 └────┬─────┘  └────┬─────┘
      │ wg0 (overlay, three logical subnets multiplexed on one interface)
      │   ├─ 100.64.0.0/16   control subnet   (node-agent ⇄ control plane only)
      │   ├─ 100.65.0.0/16   session subnet   (renter ⇄ tenant container, userspace-proxied)
      │   └─ (operator console rides the control subnet, ACL-gated)
      │
      └─ HOME LAN (192.168.x.0/24)  ── NEVER routed onto the overlay; egress-firewalled
```

Key property: a node has exactly **one** overlay interface (`wg0`) but traffic is partitioned
into logical subnets enforced by **ACLs at the coordination server** and by **WireGuard
`AllowedIPs` cryptokey routing** on each peer (§A7). The home LAN sits entirely outside the
overlay and is firewalled off (`node.md` egress firewall; `security-and-compliance.md`).

---

## A3. Choosing the overlay: Tailscale vs headscale vs Nebula vs raw WireGuard

All four candidates share the WireGuard data plane (or an equivalent). They differ in the
**control plane** — how peers discover each other, exchange keys, traverse NAT, and fall back
to relay — and in **who operates / owns** that control plane. For a fleet of thousands of
unattended nodes that must work behind CGNAT, the control plane *is* the product.

| Dimension | Tailscale (SaaS) | Self-hosted **headscale** | **Nebula** (Slack/Defined) | Raw WireGuard + custom coord |
|---|---|---|---|---|
| Data plane | WireGuard (userspace `wireguard-go` or kernel) | WireGuard (same client) | Custom (Noise/WG-like UDP) | Kernel WireGuard |
| NAT traversal | Best-in-class STUN + hole-punch | Same client logic as Tailscale | Good UDP hole-punch via lighthouses | **You build it** |
| Relay fallback | DERP (Tailscale-run + self-host) | DERP (self-host your own) | Relay via designated hosts | **You build it (TURN)** |
| Identity / ACL | Centralized, rich ACL policy | Same ACL model, self-run | Cert-based (CA), host groups via cert fields | Whatever you write |
| Key rotation / revocation | Built-in, automatic | Built-in | Re-issue/expire certs; CRL-ish | Manual / bespoke |
| Ops burden | Lowest (SaaS) | Medium (run headscale + DERP) | Medium (run lighthouses + CA) | **Highest** |
| Cost model | Per-device/seat SaaS fee | Infra only (free OSS) | Infra only (free OSS) | Infra only |
| License | Clients BSD-3; coordination proprietary | **headscale = BSD-3** (OSS reimpl of coord) | **Nebula = MIT** | WireGuard GPLv2/various |
| Lock-in | High (proprietary coordinator) | Low (OSS, self-run) | Low | None |

### A3.1 Disqualifying analysis

- **Tailscale SaaS** is the fastest to a working PoC and has the best hole-punching in the
  industry — but **per-device pricing across thousands of nodes becomes a structural opex
  line**, the coordinator is proprietary (a hard dependency we don't control), and our nodes
  are not "user devices" — they're appliances we manufacture. It is excellent for
  **internal/operator use and early pilots**, not for the production fleet data path.
- **Raw WireGuard + custom coordination** gives total control and zero license cost, but it
  means **building STUN, hole-punching, relay/TURN, key distribution, ACLs, and roaming
  ourselves.** That is a multi-engineer-year project that re-invents exactly what
  headscale/Nebula already ship. Not justified.
- **Nebula (MIT)** is genuinely strong: cert-based identity maps cleanly to "manufactured
  appliance gets a cert at provisioning," host-group ACLs via cert fields, lighthouses for
  discovery, and built-in relay. Its weakness for us: the **certificate revocation story is
  weaker** (no first-class CRL/short-lived-cert tooling until recent versions; you manage
  expiry yourself), and the ecosystem of off-the-shelf "remote console / userspace proxy"
  tooling is thinner than the WireGuard-client world.

### A3.2 Recommendation

**Self-hosted headscale as the coordination server, with the standard WireGuard/Tailscale
client (`tailscaled` in its open-source form) on nodes, and self-hosted DERP relays.**
Rationale:

1. **License + cost:** headscale is BSD-3 (free), DERP relays are self-hostable, and the
   client is open source — **no per-device SaaS fee** across the fleet. Opex is just
   relay/coord infra, which we'd pay anyway.
2. **Best NAT traversal we can get without building it:** we inherit Tailscale's mature STUN
   + hole-punch + DERP logic, which is the single most valuable thing here given CGNAT
   prevalence.
3. **We own the control plane:** headscale runs in *our* cloud next to the control plane
   (`platform.md`), so node identity, ACLs, and revocation are ours — no external dependency
   holding the fleet hostage (mirrors the overview's concentration-risk philosophy).
4. **Migration path:** because the client is interchangeable with Tailscale SaaS, we can run
   **Tailscale SaaS for the operator tailnet and early pilots** (low ops burden) and
   **headscale for the production node fleet** (low cost), same client binary on both.

**Nebula is the documented fallback** if headscale's single-coordinator scaling or DERP
operations prove painful at 10k+ nodes; its MIT license and cert model are clean. We keep the
node agent's tunnel logic abstracted (`node.md`) behind an internal `OverlayProvider`
interface so the data plane can be swapped without touching the agent's job/console logic.

---

## A4. NAT traversal mechanics — and why CGNAT is the villain

WireGuard itself does no NAT traversal; the coordinator does. The sequence to establish a
**direct** path between two peers behind NAT:

1. Both peers send keep-alive/binding packets to a **STUN** function on the coordination/DERP
   server, learning their own *public* `IP:port` as seen from outside (the "server-reflexive"
   address).
2. The coordinator relays each peer's candidate addresses (LAN address, server-reflexive
   address) to the other peer.
3. Both peers simultaneously fire UDP packets at each other's server-reflexive address
   (**hole punching**). If the NATs assign predictable ports, each NAT now has an outbound
   mapping that lets the other's packets in → **direct path established**.
4. WireGuard's roaming then locks onto whichever address the verified handshake arrived from;
   subsequent traffic is direct, encrypted, no server in the path.

### A4.1 Why CGNAT often defeats this

Hole-punching needs each side's NAT to (a) be reachable from the other's punched packets and
(b) preserve port mappings predictably. CGNAT breaks both:

- **Endpoint-Dependent Mapping (symmetric NAT):** the carrier allocates a *different*
  external port for each destination, so the port a peer learns via STUN (toward the STUN
  server) is **not** the port the carrier will use toward the other peer. The punch lands on
  the wrong port. This is the dominant CGNAT failure mode.
- **No inbound at all:** some CGNAT pools simply drop unsolicited inbound, even to a
  freshly-created mapping, if the source differs from the mapping's destination.
- **Double NAT** (home router behind CGNAT) compounds mapping unpredictability.

When direct fails, traffic falls back to **DERP/relay** (TURN-equivalent): both peers hold an
outbound connection to a shared relay, and the relay forwards encrypted WireGuard packets
between them. This **always works** (it's pure outbound for both sides) but costs us
bandwidth and adds latency.

### A4.2 Maximizing the direct-path hit rate

Relays are the opex enemy (§A6), so we engineer aggressively for direct paths:

- **UDP keepalives** (`persistent-keepalive ~25s`) on every node to hold NAT mappings open —
  without them, idle mappings expire and the next session re-relays before re-punching.
- **IPv6 first (§A9).** If both peers have working IPv6, there is *no NAT* — direct connection
  is trivial. Many "CGNAT" homes have native IPv6; preferring v6 candidates can recover a
  large fraction of otherwise-relayed nodes. This is the single highest-leverage mitigation.
- **Port-restricted hole-punch retries + birthday-paradox port prediction** for symmetric NAT
  (inherited from the Tailscale client).
- **UPnP-IGD / NAT-PMP / PCP best-effort** on non-CGNAT home routers to request an explicit
  port mapping (cheap when it works; silently ignored when it doesn't). Never *required* —
  outbound-only remains the floor.
- **Per-node NAT classification telemetry.** On boot the agent runs a NAT-type probe
  (full-cone / restricted / port-restricted / symmetric / CGNAT-hard) and reports it to the
  control plane (`platform.md`). This feeds:
  - the **reliability/network score** (a hard-CGNAT node is rated lower for data-heavy work),
  - **placement** (the scheduler co-locates data-heavy jobs on direct-capable nodes; routes
    bulk to caches for the rest — see Part B),
  - **relay-cost forecasting** (§A6).

Target: drive the **relayed-session fraction below ~15–20%** of node-hours, with the residual
concentrated on cellular-failover and hard-CGNAT nodes where we *expect* and *price* relay
cost.

---

## A5. Connection-establishment sequence (node boot → ready)

```
NODE BOOT
  │
  1. node-agent starts (node.md). Reads its WireGuard private key
  │   from TPM-sealed / secure storage (§A8). Never on disk in plaintext.
  │
  2. Outbound TLS dial → COORDINATION SERVER (headscale).
  │   Authenticates with node identity (signed pre-auth key + hardware attestation,
  │   §A8 / security-and-compliance.md). NO inbound port is ever opened.
  │
  3. Coordinator validates attestation → admits node → assigns stable overlay IP
  │   (100.64.x.x on the control subnet) + pushes ACL-filtered peer map
  │   (which peers this node may talk to: control plane, its relays; NOT other nodes).
  │
  4. NAT probe: agent contacts STUN function, classifies NAT type, learns
  │   server-reflexive addr. Tries IPv6 candidate first.  ──► report to platform.md
  │
  5. wg0 comes up. persistent-keepalive holds the mapping. Node now reachable
  │   over the overlay by control plane + operators (ACL-gated). NODE IS "ONLINE".
  │
  ── (node sits idle on overlay, heartbeating; data plane dormant) ──
  │
  6. Scheduler assigns a renter job → control plane authorizes a SESSION:
  │   pushes a scoped, time-boxed peer entry to BOTH the node and the renter
  │   (session subnet 100.65.x.x), with AllowedIPs limited to the tenant container.
  │
  7. Renter peer + node peer attempt DIRECT path:
  │     a. exchange candidates via coordinator
  │     b. simultaneous UDP hole-punch
  │     c. success → DIRECT (cheap, low-latency)  ──┐
  │        failure → fall back to DERP RELAY        ├──► path type reported as link telemetry
  │     (continuously re-attempts upgrade direct⇄relay in background)
  │
  8. Userspace proxy on the node exposes ONLY the tenant container's SSH/Jupyter/
  │   API port onto the session subnet (§A7). Renter connects. SESSION READY.
  │
  9. On job end / preemption: session peer entry expires + is revoked; userspace
      proxy closes; bulk artifacts pushed to REGIONAL CACHE not over home uplink (Part B).
```

The same machinery serves the **operator remote console** (step 6–8 but on the control
subnet, gated to the operator group, exposing a host-side diagnostic shell rather than the
tenant container — §A7.3).

---

## A6. The relay-bandwidth cost problem (a major opex line)

Direct paths cost us nothing (peer-to-peer). **Relayed paths route every byte through our
DERP POPs, which we pay for** — egress and transfer. Because consumer uplinks are slow, a
relayed *bulk* transfer is doubly bad: slow *and* metered against us. This is an explicit risk
to quantify, and it is the economic justification for the entire cache strategy in Part B.

### A6.1 Cost estimation sketch

Let:

- `N` = number of active nodes
- `r` = fraction of node-hours whose session runs over a **relay** (target ≤ 0.15–0.20 after
  §A4.2 mitigations)
- `B` = average sustained relayed throughput per relayed session (bytes/s) — bounded by the
  consumer uplink, ~1–4 MB/s
- `h` = relayed session-hours per node per month
- `C` = cloud egress/transfer price (~$0.05–0.09/GB on commodity clouds; far less on
  bandwidth-priced providers or owned transit)

Monthly relay bytes ≈ `N × r × h × 3600 × B`. Monthly relay cost ≈ that ÷ 1e9 × `C`.

**Worked example.** `N = 5,000`, `r = 0.18`, `h = 120` relayed session-hours/node/mo,
`B = 2 MB/s`, `C = $0.07/GB`:

```
relay GB/mo ≈ 5000 × 0.18 × 120 × 3600 × 2e6 / 1e9
            ≈ 5000 × 0.18 × 120 × 7.2 GB
            ≈ 777,600 GB/mo  (≈ 778 TB)
relay cost  ≈ 777,600 × $0.07 ≈ $54,400 / month
```

That is the **headline number that justifies the entire direct-path + cache strategy.**
Sensitivity: halving `r` (better hole-punching / IPv6) roughly halves cost; moving bulk pulls
off the relay (the caches in Part B) collapses `B` for the relayed sessions to control-only
traffic (kilobytes/s), cutting the number by an order of magnitude.

### A6.2 Mitigations (in priority order)

1. **Don't relay bulk — ever, if avoidable.** This is the biggest lever and is the central
   contract between this part and Part B. Model weights, base images, and datasets are pulled
   from **regional content-addressed caches** near the renter/node, not shoved through a home
   uplink and a relay. Result/checkpoint egress also targets the nearest cache. The relay then
   carries only **interactive control/console traffic** (tiny), not gigabytes. (See Part B
   §B1 for the cache hierarchy and §B3.4 for how the data layer stays relay-aware.)
2. **Maximize direct paths (§A4.2):** IPv6, keepalives, UPnP best-effort, NAT-aware retries →
   lower `r`.
3. **Regional relays / DERP POPs:** one POP per major geo so relayed traffic stays in-region
   (lower latency, often cheaper egress, no cross-continent transit). Nodes pick the
   lowest-RTT relay.
4. **Prefer-direct with continuous upgrade:** start a session on relay if needed for instant
   readiness, but keep attempting a direct upgrade in the background; once direct succeeds,
   drain the relay. Tailscale/headscale client does this natively.
5. **Cellular-failover nodes are relay-priced.** Commercial sites with LTE/5G backup almost
   always relay; the yield router / pricing treats their data-heavy capacity as more expensive
   (`platform.md`).
6. **Bandwidth-priced or owned transit for POPs.** Host DERP on providers with flat/cheap
   bandwidth (or peer/transit at IXes at scale) rather than hyperscaler egress rates —
   directly attacks `C`.

---

## A7. Network segmentation (control vs session vs home LAN)

Three isolation domains, enforced in depth (defense aligns with
`security-and-compliance.md`):

### A7.1 The three domains

| Domain | Subnet | Who | Carries |
|---|---|---|---|
| **Control plane** | `100.64.0.0/16` | node-agent ⇄ control plane; operators (ACL-gated) | heartbeat, telemetry, OTA, remote console |
| **Renter session** | `100.65.0.0/16` | renter peer ⇄ that node's tenant container only | SSH/Jupyter/API to the rented GPU |
| **Home LAN** | `192.168.x.0/24` | homeowner devices | **never on the overlay; firewalled off entirely** |

### A7.2 Enforcement mechanisms (each layer independently sufficient to contain a breach)

- **Coordinator ACLs (headscale policy):** the source of truth for "who may dial whom." A
  node's pushed peer map contains *only* the control plane + its relays by default. A renter
  peer is added **only** for the duration of an authorized session, scoped to that one node,
  and removed on teardown. Renters can never enumerate or reach other nodes.
- **WireGuard cryptokey routing (`AllowedIPs`):** even if a peer entry existed, `AllowedIPs`
  on the node restricts the renter peer to the tenant container's single overlay address —
  packets to anything else are dropped at the crypto layer before routing.
- **Userspace proxy, not L3 routing, for sessions (§A7.3).** The renter never gets an IP route
  *into* the node's network namespace; they get a single forwarded TCP port. There is no path
  from the session subnet to the host or the home LAN.
- **Host egress firewall (`node.md`):** the tenant container has no route to
  `192.168.x.0/24`, the thermal controller, or the host management plane. The overlay
  interface and the home LAN interface are in separate firewall zones; forwarding between them
  is denied by default.
- **Separate keys per domain role.** The node's control identity and the per-session
  credentials are distinct; compromising a session credential yields nothing on the control
  subnet.

The homeowner's network is treated as **hostile and off-limits**: Superheat originates
outbound to its own coordinators and never bridges, routes, or scans the LAN. This is both a
trust promise and an attack-surface reduction.

### A7.3 How operators get a console and renters get a session — without exposing the home

Both use the same trick: **the node initiates everything, and access is a userspace
port-forward over the overlay, not an inbound listener.**

- **Operator remote console.** When an operator (or an automated recovery action from the
  overview's recovery ladder) needs a node, the control plane authorizes the operator's
  overlay identity to reach that node's **host diagnostic endpoint** on the control subnet.
  The node-agent exposes a constrained shell / diagnostic API (not raw root SSH) bound to
  `wg0` only. Because the node dialed out to the coordinator and the operator is another
  overlay peer, the console works through CGNAT with zero inbound ports. This is the "reverse
  tunnel" referenced in the overview's recovery ladder.
- **Renter session.** The node-agent runs a **userspace proxy** that maps a session-subnet
  `IP:port` to the **tenant container's** SSH/Jupyter/API port inside the isolation boundary —
  and *nothing else*. The renter, also an overlay peer (joined for that session), connects to
  that port as if the GPU were local. They get `ssh renter@<overlay-ip>` or a forwarded
  Jupyter URL; they neither know nor can reach that it's a basement in someone's house. On
  preemption/teardown the proxy and the peer entry vanish.

---

## A8. Identity & key management

Node identity is the root of trust for the whole overlay; it must be issued at manufacture,
stored securely, rotated routinely, and revocable instantly.

### A8.1 Provisioning (key issuance)

- At factory/provisioning, each node generates its WireGuard keypair **on-device**; the
  private key is **sealed to the node's TPM / secure element** and never leaves it (coordinates
  with `node.md` immutable-OS + attestation).
- The node is registered in the control-plane registry with its **public key + hardware
  identity + SKU/tranche tag** (overview registry). A short-lived, single-use **pre-auth key**
  authorizes its first join.
- **Attestation gate:** before the coordinator admits a node, it verifies the node is running
  signed Superheat software (image + agent measurement) — `security-and-compliance.md`
  attestation. A node that fails attestation gets *no* overlay membership and therefore *no*
  paying work.

### A8.2 Key rotation

- **Routine rotation** of node overlay keys on a schedule (e.g., quarterly) and of **session
  credentials per session** (ephemeral by construction).
- Rotation is **make-before-break**: the agent registers the new public key with the
  coordinator, confirms the new handshake works, then retires the old key — no offline window
  for an unattended node.
- WireGuard's roaming + the coordinator's peer-map push make rotation transparent to the data
  plane.

### A8.3 Revocation (compromised or decommissioned node)

- **Instant revocation = remove the node's public key from the coordinator's policy + push
  updated peer maps.** Within one push cycle, no peer will complete a handshake with the
  revoked key; the node is cut off from the entire overlay (control, console, sessions).
- Triggers: failed attestation, anomalous behavior (`security-and-compliance.md`),
  decommission/RMA, or operator action.
- A revoked/decommissioned node also has its registry entry retired and its tranche/financing
  tag updated (overview registry + refresh-reserve modularity — a GPU module swap re-provisions
  a *new* identity).
- We keep a **revocation list / short key lifetimes** so even a node that's offline at
  revocation time cannot rejoin when it returns: its key is already gone from policy, and its
  next dial-out is rejected.

---

## A9. IPv6 considerations

IPv6 is not a nice-to-have here; it is a **primary direct-path enabler and relay-cost
reducer**.

- **No NAT on IPv6.** Where both peers have working native IPv6, hole-punching is unnecessary
  — a direct, addressable path exists. Preferring IPv6 candidates recovers many nodes that
  would otherwise relay over CGNAT'd IPv4. (§A4.2)
- **CGNAT homes frequently still have native IPv6.** Carriers that CGNAT IPv4 often hand out
  real IPv6 prefixes. Detecting and using v6 is the cheapest CGNAT mitigation available.
- **Dual-stack everywhere:** coordinator, DERP relays, and regional caches must be reachable
  over both v4 and v6 so the client can pick the working family per peer pair.
- **Firewall caveat:** native IPv6 means the node *could* be directly addressable inbound — we
  still **firewall inbound by default** and rely on the overlay's authenticated handshakes;
  IPv6 is used for *outbound-initiated* direct paths, not to open the node up. Outbound-only
  remains the principle even with public v6.
- The NAT-probe (§A4.2) reports v6 capability per node to `platform.md`; high-v6 regions are
  expected to show lower relay fractions and better network scores.

---

## A10. Observability — link quality feeding the reliability score

The overlay is also a **measurement instrument**. Every node continuously emits link telemetry
that the control plane (`platform.md`) turns into the network component of the
reliability/uptime score.

Per-node, per-session metrics:

- **Path type:** direct vs relay (and which relay POP). A persistently-relayed node is scored
  lower for data-heavy work.
- **Latency / jitter:** RTT to coordinator, to relay, and to active peers; jitter under load.
- **Upload / download bandwidth:** measured real throughput (not the ISP's advertised rate) —
  the number Part B's bandwidth-aware placement and the scheduler actually use.
- **NAT classification** (§A4.2) and **IPv6 capability** (§A9).
- **Handshake health:** churn, re-key failures, mapping expirations (early warning of a
  flapping link).
- **Keepalive / online continuity:** how reliably the node holds its overlay presence — the
  connectivity input to uptime %.

These feed three consumers (`platform.md`): the **reliability score** (which gates premium vs
spot work and feeds the securitization tape), the **scheduler/placement** (co-locate
data-heavy jobs on direct, high-upload nodes), and the **relay-cost forecast** (§A6). A node
with hard-CGNAT + low upload is perfectly fine for compute-bound interruptible work staged via
caches, and is priced/scheduled accordingly rather than penalized out of the fleet.

---

## A11. Resilience when coordination servers are down

The coordinator is a control-plane dependency, not a data-plane one — by design, an outage
degrades gracefully:

- **Cached peer config.** Each node caches its last-known peer map + keys. **Existing overlay
  sessions and direct paths keep working** with no coordinator, because WireGuard data flows
  peer-to-peer once established. Active renter sessions and operator consoles survive a
  coordinator blip.
- **What degrades:** *new* peer admission, NAT-traversal coordination for *new* peer pairs,
  ACL changes, and revocations cannot be processed until the coordinator returns. New session
  setup that needs a fresh hole-punch may stall or fall back to relay (relays can be
  cached-known too).
- **Relays are independent.** DERP POPs are separate from the coordinator; cached relay
  endpoints keep relayed paths alive during a coordinator outage.
- **HA coordinator.** Run headscale with a replicated Postgres backend (shared with the
  registry) and multiple stateless coordinator instances behind a load balancer, multi-region.
  Treat it as control-plane-critical infrastructure.
- **Revocation safety vs availability tradeoff:** because revocation needs the coordinator, we
  keep **key lifetimes short** so a node that should be cut off cannot rely indefinitely on
  cached config — bounding the window in which a coordinator outage delays a revocation.
- **Local safety is never network-dependent:** if connectivity is fully lost, the node falls
  back to operating as an ordinary water heater with compute disabled — the customer always
  has hot water (Design principle 2).

---

# Part B — Data I/O & Renter UX

## B0. The problem in one number

A residential link is asymmetric. A "good" US home plan is ~500 Mbps **down** / ~20 Mbps
**up**; a common one is ~100/10; a weak DSL/fixed-wireless node is ~25/3. The GPU is fast
(an RTX 4090 fine-tunes a 7B LoRA in minutes), but **the data around the GPU moves at the
speed of the slowest residential uplink.**

> **Worked example — a 40 GB checkpoint.**
> 40 GB = 320 Gbit (320,000 Mbit). At an effective 16 Mbps upload (a 20 Mbps plan after
> overhead/contention): `320000 / 16 ≈ 20,000 s ≈ 5.6 hours`. At 8 Mbps effective it is
> **~11 hours**. At 3 Mbps (DSL) it is **~30 hours** — longer than most thermal windows.
>
> Now flip it: that same 40 GB pushed to a **regional PoP over a 1 Gbps backhaul** (PoPs
> are colocated, not residential) takes `320000 / 800 ≈ 400 s ≈ 7 minutes`. The job only
> ever needs to reach the *nearest* fast endpoint; the slow last mile is the home uplink to
> the PoP, not the home uplink to the renter's laptop on another continent.

Three consequences drive every design decision in this part:

1. **Never pull big artifacts over a home uplink if a nearby cache has them.** (§B1)
2. **Never lose hours of work to a preemption.** Checkpoints are mandatory and must land on
   *local NVMe first*, uploaded asynchronously. (§B2)
3. **Place data-heavy work where the bandwidth is.** (§B3)

The five overview principles map directly (see the shared Design-principles section above):
outbound-only shapes how caches and checkpoints are pulled/pushed (node dials the PoP);
fail-safe-to-heater means data I/O never blocks the heater or the watchdog; thermally coupled
means I/O budgets are sized against the thermal window length from the utilization engine
(`platform.md`); measure-then-price means per-node bandwidth is a first-class metric feeding
the reliability score (`platform.md`); hybrid demand means the same data layer serves
marketplace jobs and our own backlog identically.

---

## B1. Regional content-addressed cache + pre-staging

### B1.1 Content addressing & fleet-wide dedup

Every cacheable artifact — base container image layer, model weight file, dataset shard — is
identified by the **SHA-256 of its content**, not by a name or URL. An artifact is an
immutable blob keyed `cas:sha256:<digest>`; a human name (`meta-llama/Llama-3.1-8B`,
`pytorch/2.4-cuda12.4`) is just a *manifest* — an ordered list of blob digests plus metadata.

Why this matters on this fleet:

- **Dedup across the fleet.** Ten thousand jobs that all use the same CUDA base layer or the
  same 16 GB checkpoint store *one* copy per cache tier, not ten thousand. Marketplace jobs
  (vast.ai, RunPod) and our own backlog share the same blobs.
- **Integrity for free.** A blob is verified by re-hashing on arrival; a corrupted transfer
  over a flaky residential link is detected, not silently used.
- **Cheap diffing for checkpoints.** Chunk a large artifact (content-defined chunking, ~4 MB
  average via a rolling hash) and address each chunk. A new checkpoint that is 95% identical
  to the last one only ships the changed chunks — critical when the uplink is the bottleneck
  (§B2.4).

### B1.2 The cache hierarchy

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │  ORIGIN          model hubs (HF), container registries, renter buckets │
   │  (cold, far)     S3/GCS object storage                                 │
   └───────────────────────────────┬──────────────────────────────────────┘
                                    │  pulled ONCE per region, on PoP backhaul
                                    │  (fast, colocated — never a home uplink)
   ┌──────────────────────────────────────────────────────────────────────┐
   │  REGIONAL PoP CACHE   content-addressed blob store, 10–100 TB NVMe     │
   │  (warm, near)         1 region ≈ metro/state; 1–10 Gbps backhaul       │
   │                       holds: popular base images, hot model weights,   │
   │                       shared datasets, recent checkpoints (§B2)        │
   └───────────────┬──────────────────────────────────┬───────────────────┘
                   │  LAN-speed if same site;          │  reverse-tunnel /
                   │  PoP→home over residential DOWN   │  direct mesh (Part A)
                   │  (fast direction — 100–500 Mbps)  │
   ┌───────────────▼──────────────────────────────────▼───────────────────┐
   │  ON-NODE NVMe CACHE   ≥1 TB scratch per GPU (node.md hardware floor)   │
   │  (hot, local)         holds: this node's working set —                 │
   │                       active base image, current weights, scratch,     │
   │                       in-flight checkpoint. Eviction = LRU + pinning   │
   └───────────────────────────────┬──────────────────────────────────────┘
                                    │  NVMe / PCIe — ~GB/s, never the bottleneck
   ┌──────────────────────────────────────────────────────────────────────┐
   │  GPU VRAM (24 GB)     the actual compute                               │
   └──────────────────────────────────────────────────────────────────────┘
```

Key asymmetry exploit: **a home node's DOWN link is 10–50× its UP link.** Pulling weights
*down* from a PoP is cheap; pushing results *up* is expensive. The hierarchy is designed so
that the slow (upload) direction is minimized and the fast (download) direction does the heavy
lifting.

### B1.3 Eviction & warming

**On-node NVMe (≥1 TB/GPU):**

- **Policy:** LRU with **pinning**. The currently-running job's working set is pinned and
  never evicted under it. Everything else (last job's base image, a previously popular model)
  is LRU-evicted to make room.
- **Reserve:** a fixed slice (e.g., 150 GB) is reserved for **checkpoint scratch** so a
  checkpoint can always be written locally even when the cache is full — fail-safe (§B2.2).
- **Pre-warming:** when the scheduler (`platform.md`) tentatively assigns a job to a node, it
  issues a **prefetch hint** so the node pulls the job's manifest blobs from the PoP *down*
  link during the tail of the previous job / pre-heat window from the utilization engine
  (`platform.md`). Goal: GPU starts computing within seconds of job start, not after a
  multi-minute pull.

**Regional PoP cache (10–100 TB):**

- **Policy:** weighted LRU-K — frequency × recency × size, biased to keep small, very-hot
  blobs (popular base layers, top-20 open models) resident and evict large cold datasets
  first.
- **Origin warming:** a control-plane job watches the demand stream and pre-pulls trending
  artifacts (e.g., a newly released model) into every active region *before* the demand wave,
  on PoP backhaul. One origin pull per region, amortized over thousands of node pulls.
- **TTL on checkpoints:** see §B4 lifecycle table — checkpoints in the PoP are short-lived.

### B1.4 How a job actually pulls weights

1. Scheduler assigns job → node, with the job manifest (list of blob digests).
2. Node agent diffs manifest against on-node NVMe cache → list of **missing chunks**.
3. For each missing chunk, agent requests it from the **nearest PoP** over the overlay
   (Part A), preferring a **direct mesh path** to the PoP and the *download* direction.
4. PoP serves from its NVMe; on a PoP miss, the PoP (not the node) pulls from origin once and
   caches it, then serves the node.
5. Chunks verified by hash, assembled, job container starts.

The renter's slow uplink is **never** on this path. The renter uploads their model/dataset to
the PoP/origin once (or references a public artifact); from then on it is cache-served.

---

## B2. Checkpoint / resume — mandatory

Jobs on standard nodes **will** be preempted: the thermal window ends, or the yield router
(`platform.md`) hands the node a higher-paying job. Checkpoint/resume is what makes
interruptible filler and reliability tiering safe to sell. It is not optional plumbing; it is
the contract.

### B2.1 Recovery-Time Objective (RTO)

> **RTO target: a preempted job resumes useful compute on a new node within 5 minutes
> (p50 ≤ 2 min, p95 ≤ 10 min) for working sets ≤ 40 GB**, given the checkpoint is already at
> a PoP. RPO (lost work): ≤ one checkpoint interval (§B2.4), default **≤ 15 minutes of
> compute**.

Why 5 minutes is achievable despite slow uplinks: the *expensive* upload (node → PoP) happens
**asynchronously during the job**, not at preemption time. At resume, the new node *downloads*
the checkpoint from the PoP over its fast down-link. Preemption itself only needs to flush the
last delta to local NVMe and signal the control plane.

### B2.2 Three-stage checkpoint, decoupled from preemption

```
   Stage 1: WRITE to local NVMe        (fast, always succeeds — fail-safe)
       └─ torch.save / framework hook → /scratch/ckpt/<job>/step-<n>/
       └─ completes in seconds–tens-of-seconds (NVMe is GB/s)

   Stage 2: ASYNC UPLOAD to PoP        (slow, over residential UP — but in background)
       └─ content-defined chunk + dedup vs previous checkpoint (§B2.4)
       └─ only changed chunks shipped; rate-limited to not starve the renter's egress
       └─ runs CONCURRENTLY with the next compute step

   Stage 3: REGISTER in control plane  (tiny metadata write)
       └─ "job J has durable checkpoint at step n, located at pop:<region>, digest:<...>"
       └─ scheduler can now safely preempt J and resume it elsewhere
```

The critical invariant: **a job is only considered "safely preemptible up to step n" once
stage 3 for step n completes.** If preemption arrives before the latest local checkpoint has
reached the PoP, the agent either (a) gets a short grace window to finish the in-flight upload
(see §B2.3), or (b) the job resumes from the last *PoP-durable* checkpoint, losing at most one
interval (RPO).

Local NVMe write **never blocks the heater or the watchdog.** Data I/O is the lowest-priority
local consumer; `system-design-overview.md` makes local safety win over everything.

### B2.3 Checkpoint/resume sequence

```
 Renter   Scheduler        Node A (running)        PoP cache         Node B (resume)
   │      (platform.md)         │                    │                   │
   │ submit job ───►│            │                    │                   │
   │               │ assign J ──►│                    │                   │
   │               │             │ pull weights ◄─────│ (down-link, fast) │
   │               │             │ ── COMPUTE ──      │                   │
   │               │             │ ckpt step1 → NVMe  │                   │
   │               │             │ async upload ─────►│ (delta chunks)    │
   │               │ ◄── register durable(step1) ─────│                   │
   │               │             │ ── COMPUTE ──      │                   │
   │               │             │ ckpt step2 → NVMe  │                   │
   │               │             │ async upload ─────►│                   │
   │               │ ◄── register durable(step2) ─────│                   │
   │               │             │                    │                   │
   │   ⚡ PREEMPT (thermal window ends OR higher-yield job — platform.md)
   │               │ preempt J ─►│                    │                   │
   │               │             │ flush last delta ─►│ (grace ≤ 30s)     │
   │               │             │ SIGTERM container  │                   │
   │               │             │ → fall back to heater-only if needed   │
   │               │ ◄── J stopped @ durable step2 ───│                   │
   │               │                                  │                   │
   │               │ reassign J ──────────────────────────────────────►  │
   │               │                                  │ ckpt step2 ──────►│ (down-link)
   │               │                                  │ weights ─────────►│
   │               │                                  │      ── RESUME COMPUTE @ step2 ──
   │ ◄──────────── job continues, renter sees only a brief stall ────────│
```

Renter experience: a transient pause, not a failure. The session endpoint (§B5.1) reconnects
to Node B over the overlay; from the renter's CLI it looks like a reconnect.

### B2.4 Frequency vs overhead tradeoff

Checkpointing costs compute time (serialize + the I/O steal) and uplink bandwidth. Too rare =
big RPO loss on preemption; too frequent = overhead dominates.

- **Adaptive interval, driven by preemption risk from the utilization engine (`platform.md`).**
  The thermal coordinator publishes an expected-remaining-window estimate per node. Checkpoint
  cadence is set so the *expected work lost on preemption* (RPO) stays bounded:
  - Stable node, long window → checkpoint every ~15–30 min.
  - Node near end of thermal window or flagged for imminent higher-yield preemption →
    checkpoint *now* (event-driven, before the SIGTERM lands).
- **Delta + dedup is what makes this affordable on a slow uplink.** A 40 GB training state
  that changes ~2 GB/checkpoint ships **2 GB**, not 40 GB. At 16 Mbps that delta is ~17 min —
  still significant, hence: (a) checkpoints upload *concurrently* with compute, and (b)
  cadence is matched to the upload time so a new checkpoint doesn't start before the prior
  upload finishes. If the uplink can't keep up, the scheduler treats the node as **low
  checkpoint-bandwidth** and routes only small-state jobs there (§B3).

### B2.5 Framework integration & the GPU-state limit

- **Preferred: application-level checkpointing.** PyTorch `state_dict` (model + optimizer +
  scheduler + RNG + step), HF `Trainer` resume-from-checkpoint, DeepSpeed/FSDP sharded
  checkpoints, Lightning callbacks. This is **small, portable, and resumes on different
  hardware** — exactly what cross-node resume needs. We ship a thin `superheat-ckpt` library /
  callback that points the framework's save path at `/scratch/ckpt/...` and triggers stage 2
  on save. This is the path we require for our own backlog and recommend to renters.
- **Container snapshot (CRIU): limited.** CRIU can checkpoint a process tree's CPU/memory
  state, but **it cannot reliably snapshot live GPU/CUDA context** (device memory, streams,
  driver state) on consumer drivers. NVIDIA's `cuda-checkpoint` enables a flow (drain device
  state to host → CRIU → restore), but it is fragile across driver/hardware mismatches and is
  large (full device memory dump, up to 24 GB). **Decision: CRIU/cuda-checkpoint is a
  best-effort fallback for opaque/legacy renter jobs that don't checkpoint themselves —
  resume is only attempted on the *same* node SKU, and the large snapshot stays as a local
  NVMe checkpoint with best-effort async upload.** It is never the primary mechanism.
- **Honest limit stated to renters:** seamless preempt/resume is *guaranteed* only for jobs
  that use framework checkpointing (or our SDK). Opaque jobs get best-effort and may restart
  from scratch on preemption — and are therefore steered to high-reliability,
  non-interruptible nodes (`platform.md` tiering) or priced accordingly.

---

## B3. Bandwidth-aware placement

### B3.1 Measure every node's real bandwidth

The node agent continuously measures real up/down throughput and feeds it to `platform.md`,
where it becomes part of the **reliability score** (alongside uptime and thermal
availability). Measurement blends passive (observed transfer rates during real cache pulls and
checkpoint uploads — free, always-on) and active (periodic lightweight probe to the nearest
PoP — used to refresh idle nodes), so we never waste a renter's metered uplink on synthetic
tests during a paid job. This is the same observability discipline as Part A §A10 — measured
throughput, never advertised rate.

### B3.2 Per-node bandwidth metric schema

```jsonc
{
  "node_id": "h1-7a3f9c2e",
  "region": "us-east-oh",
  "measured_at": "2026-06-18T14:02:11Z",
  "link": {
    "down_mbps_p50": 280.4,        // sustained download, passive+active blend
    "down_mbps_p95": 320.0,
    "up_mbps_p50": 14.8,           // THE constraint — drives placement
    "up_mbps_p95": 18.1,
    "up_mbps_floor_24h": 9.2,      // worst sustained uplink seen in 24h
    "rtt_ms_to_pop": 11.3,
    "jitter_ms": 4.1,
    "loss_pct": 0.2
  },
  "path": {
    "to_pop": "direct-mesh",       // direct-mesh | relay  (Part A)
    "relay_fallback_active": false,// true => bulk transfers are COSTING us $$
    "cgnat": true,
    "nearest_pop": "pop-us-east-oh-1"
  },
  "cache": {
    "nvme_total_gb": 1024,
    "nvme_free_gb": 412,
    "ckpt_reserve_gb": 150,
    "resident_blobs": 1873,        // for dedup/affinity decisions
    "warm_models": ["llama-3.1-8b", "sdxl-1.0"]
  },
  "checkpoint": {
    "effective_upload_mbps": 12.1, // up throughput net of rate-limiting
    "max_state_gb_for_rto": 38,    // largest working set that still meets §B2.1 RTO
    "avg_delta_gb": 1.9,           // typical checkpoint delta (informs cadence)
    "last_ckpt_lag_s": 47          // local-write → PoP-durable lag
  },
  "scores": {
    "bandwidth_tier": "B",         // A/B/C/D — feeds reliability score & router
    "data_heavy_ok": false,        // can it host big-IO jobs? (A-tier only)
    "egress_cost_flag": "ok"       // ok | relay-expensive
  }
}
```

This schema is the contract between the data layer and `platform.md` (see also
`data-contracts.md`). `up_mbps_floor_24h` and `max_state_gb_for_rto` are the two fields the
scheduler reads most: a node's *worst* uplink and the biggest checkpoint it can keep durable
inside the RTO.

### B3.3 Placement rules

The scheduler in `platform.md` applies, on top of thermal/reliability constraints:

1. **Co-locate data and compute.** Prefer a node whose on-node cache already holds the job's
   manifest blobs (cache affinity) — zero pull cost. Then prefer a node in the **same region**
   as the dataset's PoP (down-link only). Avoid placing a 200 GB-dataset job on a node that
   would have to stream it cross-region.
2. **Match working-set size to uplink.** Big-checkpoint / data-heavy jobs → **A-tier
   bandwidth** nodes (`data_heavy_ok: true`, high `max_state_gb_for_rto`). Small-state jobs
   (LoRA fine-tunes, inference) → anywhere, including C/D-tier uplinks.
3. **Route bulk transfers over direct mesh, not paid relays.** Part A flags when a node is on
   a **relay/TURN** fallback (CGNAT hole-punch failed). Relay bandwidth is a real opex line
   (Part A §A6). Rules: (a) never schedule a *data-heavy* job onto a relay-only node if a
   direct-path node is available; (b) when a relayed node must run a big job, prefer pulling
   from a PoP reachable over a direct path; (c) the `egress_cost_flag` lets the yield router
   subtract expected relay cost from that node's $/GPU-hr so the economics stay honest.
4. **Result egress shaping.** Result/checkpoint uploads are rate-limited below the link's
   `up_mbps_floor_24h` so they never saturate the homeowner's connection (anti-abuse, keeps
   the host happy — `security-and-compliance.md`, and a fail-safe-to-customer concern).

### B3.4 Staying relay-aware (the data ↔ connectivity contract)

Part A owns the overlay and the relay budget; Part B owns *what* moves and *how much*. The
shared rule: **the data layer is the biggest consumer of relay bandwidth, so it must be
relay-aware.** Caching (PoP-served, mostly down-link) and checkpoint dedup (ship deltas, not
full state) exist substantially to keep traffic *off* paid relays — collapsing the `B`
(throughput-per-relayed-session) term in the Part A §A6.1 cost model down to control-only
traffic. Any future relay-budget alarm (Part A §A6.2) should first throttle data I/O (the
elastic load), never control/heartbeat traffic.

---

## B4. Renter UX

### B4.1 One-command entry over the overlay

The renter never learns the GPU is in a basement behind CGNAT. They get a normal-looking
endpoint, brokered through the overlay (Part A), outbound-initiated by the node.

```bash
# Container workflow
$ superheat run --gpu 4090 --image pytorch/2.4-cuda12.4 \
      --volume my-project:/workspace --interruptible -- python train.py

# SSH-style interactive (mediated tunnel — no inbound port on the home router)
$ superheat ssh job-7a3f9c2e
renter@node:/workspace$   # looks like a normal box; it's a basement in Ohio
```

- `--interruptible` opts the job into the spot/filler tier (cheaper, preemptible, requires
  checkpointing per §B2). Omitting it requests a high-reliability, non-interruptible node
  (`platform.md` tiering) at a premium.
- The session is an **outbound reverse tunnel** terminated at a broker the renter connects to;
  on preemption+resume (§B2.3) the broker re-points the tunnel to the new node, so the
  renter's terminal/endpoint URL is stable across a resume.

### B4.2 Persistent volumes with CLEAR eviction semantics

Preemption must never be a surprise. Each job has explicit storage classes with a published
contract:

| Storage class | Where | Survives preemption? | Survives job end? | Renter must back up? |
|---|---|---|---|---|
| **Ephemeral scratch** | on-node NVMe | ❌ (wiped on node change) | ❌ | n/a — treat as disposable |
| **Checkpoint dir** (`/scratch/ckpt`) | NVMe → PoP (async) | ✅ via PoP (§B2) | retained `T_ckpt` then GC'd | no (auto-managed) |
| **Persistent volume** (`--volume`) | PoP-backed, content-addressed | ✅ (re-attached on resume) | retained `T_vol` after job end | back up before `T_vol` expires |
| **Result/output egress** | renter bucket / PoP staging | ✅ | retained `T_out` | pulled by renter or auto-pushed |

Default lifetimes (configurable per contract / tier):

- `T_ckpt` (checkpoints): **24 h** after job end, then GC'd. Long enough to resume a stalled
  job, short enough to bound storage and limit data exposure.
- `T_vol` (persistent volumes): **7 days** after last detach, then GC'd with two warnings (at
  48 h and 6 h remaining) via the renter API/CLI.
- `T_out` (results): **72 h** in PoP staging, or pushed immediately to the renter's own bucket
  if configured (preferred — keeps results off our infra).

The CLI surfaces this so it is never a surprise:

```bash
$ superheat volume ls
NAME        SIZE    BACKED-BY   STATUS     GC-IN
my-project  84 GB   pop-us-e-1  attached   7d (resets on detach)
```

### B4.3 Result / output egress

The expensive direction. Two patterns:

1. **Push to renter's own object store (preferred).** Renter supplies scoped, write-only
   credentials; the node uploads results **directly** to the renter's S3/GCS over a direct
   mesh path, rate-limited (§B3.3). Keeps large outputs off Superheat infra (cost + privacy).
2. **PoP staging + pull.** Results land in the PoP (content-addressed, deduped), renter pulls
   over fast PoP down-link. Held `T_out`. Used when the renter has no bucket.

In both cases the *home uplink* moves the result once to a fast endpoint; the
renter-to-result hop is fast. Big-output jobs (e.g., video generation) are flagged
`data_heavy` and steered to A-tier uplink nodes (§B3.3) so the egress fits the window.

---

## B5. Data lifecycle & privacy hooks

Coordinates with `security-and-compliance.md`, which owns isolation/attestation and the honest
"no hardware TEE on a 4090" limit. This section owns the **data-at-rest and data-lifecycle**
guarantees the data layer can actually deliver.

- **Ephemeral by default.** On-node scratch and any pulled renter data are written to a
  **per-job, encrypted NVMe area** and are **wiped on job end** (`security-and-compliance.md`
  wipe-on-job-end). Default no-persistence: a renter must explicitly request a persistent
  volume (§B4.2) for anything to survive.
- **Encrypted at rest, per job.** Each job's NVMe area uses a **per-job key** held only in
  control-plane memory / a KMS, never persisted to the node disk. Wipe-on-end = destroy the
  key + TRIM the blocks. A node that loses power mid-job comes back with an unreadable area (no
  key) — the heater still works (fail-safe), the renter data does not leak.
- **Encrypted in transit, end to end.** All cache pulls, checkpoint uploads, and result egress
  ride the WireGuard overlay (Part A) and are additionally application-encrypted with the
  per-job key, so even a PoP stores renter checkpoints/volumes **encrypted to a key the PoP
  doesn't hold** unless the renter opts into PoP-side dedup of private data. (Public/base
  artifacts are not per-job-encrypted — that's what enables fleet-wide dedup in §B1.1. Private
  renter blobs are encrypted and dedup only within the renter's own keyspace.)
- **Lifecycle GC is the privacy backstop.** `T_ckpt` / `T_vol` / `T_out` (§B4.2) bound how
  long any renter byte lives anywhere in the system. GC = key destruction + block TRIM, logged
  to the audit trail. Short default TTLs limit the blast radius given the no-TEE reality.
- **What we do NOT promise.** Per `system-design-overview.md` and `security-and-compliance.md`,
  standard 4090 nodes cannot offer hardware-enforced confidentiality against a determined
  host. Encryption-at-rest + ephemerality + wipe-on-end raise the bar but are software
  guarantees. Sensitive workloads belong on the **trusted-host tier**; the data layer tags
  every artifact and volume with its tier so they never silently land on an untrusted node.

---

# Part C — Build phasing & open questions

## C1. Build phasing (maps to the overview roadmap)

- **Phase A (attach for revenue):** node-agent overlay client + self-hosted headscale + one
  DERP region + NAT probe + basic direct/relay + control subnet for telemetry/console.
  Tailscale SaaS for the internal operator tailnet to move fast. On the data side: one regional
  PoP cache + content-addressed pulls + local NVMe scratch. Enough to safely remote-op nodes
  and run vast.ai attach.
- **Phase B (own orchestration):** session subnet + userspace renter proxy + per-session
  credentials + full ACL segmentation + link telemetry into the reliability score + relay-cost
  dashboards. On the data side: mandatory checkpoint/resume with the `superheat-ckpt` SDK +
  bandwidth-aware placement + the per-node bandwidth schema feeding the scheduler + persistent
  volumes with published TTLs.
- **Phase C (own demand & UX):** multi-region DERP POPs + IPv6-first traversal + automated
  revocation/rotation at fleet scale + `OverlayProvider` abstraction validated against Nebula
  as a contingency. On the data side: multi-region cache fan-out + origin pre-warming on the
  demand stream + full one-command renter UX with stable sessions across resume. Bulk-off-relay
  (Part B ↔ Part A) is tightly integrated and is the assumption the relay budget rests on.

## C2. Open questions / risks

**Connectivity:**

- **headscale single-coordinator scaling** beyond ~10k nodes — validate, or shard by region /
  fall back to Nebula (§A3.2).
- **Relay opex realism** — the §A6.1 model must be tracked against actuals from day one;
  bulk-off-relay (Part B) is the assumption the whole budget rests on.
- **CGNAT hardness distribution** — we don't yet know the real direct-path hit rate across our
  actual ISP mix; the NAT-probe telemetry (§A4.2) is partly a discovery tool for this. Cold
  reality may push `r` higher than the §A6.1 target until IPv6 adoption helps.
- **Userspace-proxy throughput ceiling** — userspace WireGuard + proxying can cap per-session
  throughput; for high-bandwidth direct sessions, prefer kernel WireGuard where the immutable
  OS allows it (`node.md`).
- **Operator console blast radius** — the host diagnostic endpoint is powerful; it must be a
  constrained API with full audit (`security-and-compliance.md`), not raw root, even though it
  rides the trusted control subnet.

**Data I/O:**

- **PoP capex/opex sizing** — the 10–100 TB NVMe per region and 1–10 Gbps backhaul are
  estimates; real cache hit rates and the regional node-to-PoP ratio decide the economics
  (`financial-and-roadmap.md`).
- **Checkpoint delta realism** — the "95% identical, ship 2 GB of 40 GB" assumption (§B2.4)
  drives the whole RTO/RPO budget; if real training-state deltas are larger, slow-uplink nodes
  fall out of the interruptible tier faster than modeled.
- **CRIU/cuda-checkpoint fragility** — best-effort opaque-job resume (§B2.5) may prove too
  brittle to offer at all; if so, opaque jobs are non-interruptible-only.

---

*Cross-references:* `system-design-overview.md` (master — connectivity and data I/O sections),
`node.md` (node-agent, immutable OS, egress firewall, TPM/attestation, NVMe hardware floor,
kernel-vs-userspace WireGuard), `platform.md` (coordination server, scheduler, reliability
scoring, registry, yield router, utilization/thermal windows + preemption, renter API),
`security-and-compliance.md` (isolation, attestation, encryption, wipe-on-end, trusted-host
tier), `data-contracts.md` (the bandwidth/telemetry schemas exchanged),
`financial-and-roadmap.md` (relay + PoP cost lines, roadmap), `operations.md` (link/bandwidth
observability, incident response).
