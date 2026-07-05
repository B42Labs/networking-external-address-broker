# networking-external-address-broker

Distributed external networks on OpenStack — consolidated concept and implementation
blueprint for a Neutron IPAM plugin, a central address broker (EAB), and pluggable
floating IP realization paths across multiple OpenStack CloudPods.

> **Status:** Concept / basis for implementation.
> **Purpose:** Consolidates [EAB-CORE-DESIGN.md](EAB-CORE-DESIGN.md) (EAB core) and
> [EXTERNAL-NAT-REALIZATION.md](EXTERNAL-NAT-REALIZATION.md) (external NAT path) into one
> implementable blueprint. Where those documents leave a decision open, **this document fixes
> it**; for everything already specified there, this document references instead of repeating.
>
> Reference notation: `R§n` = EAB-CORE-DESIGN.md section n, `X§n` =
> EXTERNAL-NAT-REALIZATION.md section n.

---

## 1. The Vision

One operator-wide public address space, consumed by many autonomous OpenStack control planes
(CloudPods) as if they were a single cloud:

- **Any public IP can be used in any CloudPod — exactly once.** Allocation is conflict-free
  across all pods, enforced centrally.
- **Traffic finds the IP wherever it is realized.** Routing follows allocation via BGP; no
  static partitioning of the address space per pod is required for floating IPs.
- **Realization is a property of a router, not of the cloud.** The same allocated IP can be
  realized OVN-natively (`dnat_and_snat`) or by an external NAT layer (AWS-style), selected
  per router via an L3 flavor.
- **The public IP is a pure control-plane object.** It can be created, bound, unbound and —
  in the target picture — moved between workloads (and eventually between pods) without
  renumbering anything.
- **Not all public space is anonymous.** A range can be dedicated to one customer — usable
  only in specific domains, projects, or CloudPods — and customers can bring their own
  prefix and origin ASN (BYOIP/BYOAS, §4.3).

The architecture separates four concerns; keeping them independent is what makes the
distribution work:

| Layer | Question it answers | Owner |
|---|---|---|
| **Allocation** | Who owns the address? | EAB master (single source of truth) |
| **Association** | Which workload does it map to? | Neutron `floatingip` API (per pod, unchanged UX) |
| **Realization** | Where are packets translated? | OVN `dnat_and_snat` **or** external NAT layer |
| **Announcement** | How does the internet find it? | BGP: `ovn-network-agent` (/32) or NAT layer (/32 or aggregate) |

## 2. Document Map

| Document | Role |
|---|---|
| [EAB-CORE-DESIGN.md](EAB-CORE-DESIGN.md) | Technical reference for the EAB core: IPAM driver (R§4), agent (R§5), master (R§6), flows (R§7), error handling (R§8), operations (R§9–11). |
| [EXTERNAL-NAT-REALIZATION.md](EXTERNAL-NAT-REALIZATION.md) | Technical reference for the external realization path: L3 flavor driver (X§6.1), master NAT extensions (X§6.2), NAT layer (X§6.4), flows (X§7). |
| **README.md** (this document) | Decisions (§5), consolidated invariants and data model (§4, §7), gap closure (§6, §8–§10), milestone plan (§11). Supersedes "open" / "to be decided" statements in the other two. |

## 3. Analysis: What Is Specified vs. What Was Open

### 3.1 Specified and implementable as-is

- Allocation model with DB-enforced atomicity and race handling (R§6, R§8.3).
- IPAM driver interception of floating IP allocations only, fail-closed semantics (R§4).
- Agent design incl. cache, sync with grace period, heartbeat (R§5, R§7.3).
- Authentication/authorization model: token bound to one CloudPod (R§11).
- External NAT target architecture, packet flows, stateless NAT + BGP announce (X§4–X§7).
- Operating rule for non-FIP ports on external networks (disjoint per-pod pools, R§8.6).

### 3.2 Open until now — resolved in this document

| Open item | Where raised | Resolution |
|---|---|---|
| Reservation leak windows (failed Neutron transactions) | R§4.8, R§12.2 | **D7** — leases + confirm protocol, promoted into the core |
| Idempotent reserve | R§12.2 | **D7** — idempotency key |
| IRIP attachment variant | X§5.1 | **D4** — Option C fixed as the default |
| IRIP lifecycle granularity (per port vs. per FIP) | X§6.2 | **D8** — one IRIP per active FIP association |
| NAT layer connection (push vs. BGP-driven) | X§6.3 | **D5** — push exporter + periodic reconcile |
| Mixed pools vs. aggregate announcement | X§6.4 | **D6** — realization path is a pool property |
| L3 flavor delegation feasibility | X§6.1, X§10 | **M0 spike S1** gates the external path only |
| CloudPod offline-marking task | R§12.2 | scheduled in **M1** |
| Admin force-release | R§10.3, R§12.2 | scheduled in **M1** |

### 3.3 Missing entirely — added by this document

- Brownfield adoption: importing pre-existing FIPs before enabling the driver (§9).
- Test and verification strategy incl. multi-pod chaos tests (§10).
- API contract freeze and versioning rules (§6).
- Packaging and deployment reference (§8).
- Explicit system invariants any implementation must maintain (§4.2).
- Cross-pod FIP mobility named as the long-term target the design must not preclude (§12).
- Pool tiers: dedicated (customer-scoped) public pools and BYOIP/BYOAS (§4.3, D11–D14,
  §9.4–9.5).

## 4. Consolidated Target Architecture

### 4.1 Big picture

```
 ┌───────────────────────────── Control plane ─────────────────────────────┐
 │                                                                         │
 │                        ┌────────────────────┐                           │
 │                        │     EAB Master     │  allocations, mappings,   │
 │                        │  (HA: N replicas   │  IRIP pool, NAT export    │
 │                        │   + HA PostgreSQL) │                           │
 │                        └───┬────────────┬───┘                           │
 │            reserve/release │            │ /v1/nat/* + exporter          │
 │            confirm/sync    │            │                               │
 │   ┌────────────────────────┴──┐      ┌──┴──────────────────────────┐    │
 │   │ per CloudPod (xN)         │      │ NAT controller (per region) │    │
 │   │  EAB agent                │      │  consumes /v1/nat/mappings  │    │
 │   │  Neutron:                 │      │  programs nftables + FRR    │    │
 │   │   - EAB IPAM driver       │      └──────────────┬──────────────┘    │
 │   │   - L3 flavor driver      │                     │                   │
 │   │     "external-nat"        │                     │                   │
 │   └──────────┬────────────────┘                     │                   │
 └──────────────┼──────────────────────────────────────┼───────────────────┘
                │                                      │
 ┌─ Data plane ─┼──────────────────────────────────────┼───────────────────┐
 │              ▼                                      ▼                   │
 │  Path A (OVN-native, default)          Path B (external NAT)            │
 │  ─────────────────────────────         ────────────────────────────     │
 │  OVN dnat_and_snat (FIP↔fixed)         NAT nodes: stateless FIP↔IRIP    │
 │  ovn-network-agent announces           NAT nodes announce FIP /32 or    │
 │  FIP /32 → upstream                    aggregate → upstream             │
 │                                        OVN dnat_and_snat (IRIP↔fixed)   │
 │                                        ovn-network-agent announces      │
 │                                        IRIP /32 → DC fabric             │
 └─────────────────────────────────────────────────────────────────────────┘
```

Both paths share the entire allocation machinery; they differ only in realization and
announcement. Path selection is per router (L3 flavor, D3) and per pool (D6).

### 4.2 System invariants

Every milestone's acceptance tests assert these; any implementation change must preserve them.

- **I1 — Unique allocation.** An address from a managed pool is allocated to at most one
  CloudPod at any time. Enforced by `UNIQUE(ip_address, managed_subnet_id)` in the master DB
  (R§6.3) — never by convention.
- **I2 — Single realization point.** At any time, at most one realization (an OVN NAT entry
  *or* a NAT-layer rule set) is active per FIP, and announcements never steer one IP toward
  two owners. Guaranteed structurally by D3 + D6.
- **I3 — Fail closed.** No component allocates a managed address without the master's
  confirmation. `fallback_to_local = true` is a documented emergency override, off by default
  (R§8.1).
- **I4 — Master is authoritative.** Agent caches and NAT-layer rule sets are derived state
  and reconcile toward the master (R§7.3, X§6.3). Repair always flows master → edge.
- **I5 — Control plane failures never touch realized traffic.** Master or agent outages block
  *new* allocations/bindings only; existing NAT entries, rules, and BGP announcements stay up.
- **I6 — Scope enforcement.** An address from a `dedicated` pool is never allocated to a
  project, domain, or CloudPod outside the pool's scope. Enforced in the master's reserve
  path (D11) — Neutron RBAC only shapes visibility and is never the enforcement point.
- **I7 — Authorized announcement only.** A customer-owned prefix (BYOIP) is never exported
  to the NAT layer or announced before its authorization evidence is verified
  (`authz_state = verified`, §9.5). Enforced structurally: the exporter omits unverified
  pools (D13).

### 4.3 Address pool tiers

Managed public pools come in three tiers; higher tiers add properties, while the allocation
machinery (I1, D7) is identical for all of them.

| Tier | Prefix owned by | Who may allocate | Origin ASN | Realization |
|---|---|---|---|---|
| **1 — public** | operator | every project | operator | Path A or B (D6) |
| **2 — dedicated** | operator | pool scope only (D11) | operator | Path A or B (D6) |
| **3 — BYOIP** | customer | pool scope only (implied) | operator (**B1**, default) or customer ASN (**B2**/BYOAS) | Path B only (D13) |

Tier 2 realizes "private" public addresses: a public range reserved for one customer,
usable only in specific domains, projects, or CloudPods. Tier 3 additionally imports a
customer-owned prefix — optionally announced with the customer's origin ASN — and implies a
dedicated scope; it builds entirely on Tier 2 machinery plus the D13 announcement rules.

## 5. Design Decisions

Each decision: **what**, *why*, and consequences. These are binding for the implementation;
revisiting one requires updating this section.

### D1 — Fail-closed allocation
Allocation consistency is prioritized over allocation availability. Master unreachable ⇒ FIP
create returns 503 (R§8.1). *Why:* a duplicate public IP is a routing incident affecting
third parties; a delayed FIP creation is a retry. Consequence: master availability is an
operational SLO (mitigated by D2), and monitoring of `master_connected` is mandatory (R§10.1).

### D2 — One allocation authority, HA through the database
The master API is stateless; run ≥2 replicas behind a load balancer on HA PostgreSQL
(Patroni). Correctness comes from the DB unique constraint, not from process singularity
(R§12.2). *Why:* avoids consensus protocols in application code. Consequence: the DB is the
real availability anchor; DB failover procedures are part of the runbook.

### D3 — Realization is pluggable per router via L3 flavors
Path A (OVN-native) is the default; Path B (`external-nat` flavor) is opt-in per router
(X§6.1, R§3.4). Both are first-class and coexist indefinitely. *Why:* incremental adoption,
no big-bang migration, per-workload choice. Consequence: the flavor-framework caveat (the OVN
plugin skips flavored routers entirely) makes **spike S1 (§11, M0) the gate for Path B** —
Path A and the allocation core do not depend on it.

### D4 — IRIP attachment via Option C (OVN translates IRIP ↔ fixed IP)
The `external-nat` L3 driver writes a `dnat_and_snat` row with `external_ip = IRIP` instead
of the public FIP (X§5.1, Option C). *Why:* guest untouched, tenant CIDRs may overlap, port
security satisfied, `ovn-network-agent` reuses its NAT-entry-driven logic with only the
announcement target changed. The document goal holds: the *public* FIP never appears in OVN.
Consequence: OVN is not NAT-free on Path B — accepted trade-off; Options A/B remain documented
alternatives (X§5.1) but are not implemented.

### D5 — NAT layer driven by push + reconcile, master authoritative
The master (exporter) pushes mapping changes to the NAT controller; the controller
additionally performs a periodic full reconcile against `GET /v1/nat/mappings` (X§6.3).
*Why:* push gives low latency, reconcile gives convergence after any missed event.
Consequence: the NAT controller must be idempotent and treat the mapping list as desired
state (§6.3).

### D6 — Realization path is a property of the managed pool
Each public `ManagedSubnet` carries `realization ∈ {ovn, external-nat}` (§7). FIPs from an
`external-nat` pool are only usable on `external-nat` routers and vice versa; the master
rejects `/v1/nat/bind` for FIPs from `ovn` pools. *Why:* this is what makes aggregate-prefix
announcement from the NAT nodes safe (X§6.4) and enforces I2 structurally instead of
operationally. Consequence: operators provision (at least) two public pools when using both
paths; documented in §9.

### D7 — Reservation lifecycle: leases, confirm, idempotency (promoted into the core)
R§12.2 listed leases and idempotent reserve as future work; they are **required** for
correctness (the leak window in R§4.8 is otherwise only mitigated by the sync grace
heuristic). Fixed protocol:

1. `reserve` creates the allocation with `state = pending` and
   `lease_expires_at = now + lease_ttl` (default 300 s). If the caller supplies an
   `idempotency_key` (the FIP port ID when available at `get_request` time — verify during
   M2), a repeated reserve with the same key returns the existing reservation instead of a
   new one.
2. **Confirm** turns `pending` into `active` (clears the lease). Two independent triggers:
   a new `AFTER_CREATE` floating-IP callback in the plugin (post-commit, analogous to the
   existing `AFTER_DELETE` callback, R§4.7) calls `POST /v1/addresses/confirm`; and the
   agent's sync loop confirms any pending reservation it observes as an existing FIP in
   local Neutron.
3. A master-side **reaper** (runs with `cleanup.orphan_check_interval`) releases `pending`
   allocations whose lease expired — this deterministically closes the failed-transaction
   leak window.
4. The agent-side release-on-absence rule (R§7.3) now applies to `active` allocations only;
   `pending` ones belong to the reaper. The grace period stays as defense in depth.

State machine: `PENDING ──confirm──▶ ACTIVE ──release──▶ (gone)`;
`PENDING ──lease expiry──▶ (gone)`.

### D8 — One IRIP per active FIP association
An IRIP is allocated at `nat/bind` time and bound to the mapping record (FIP ↔ port), not to
the port's lifetime (X§6.2 left this open). On unbind, the IRIP returns to the pool. On FIP
re-association, the new bind allocates a fresh IRIP; the exporter replaces the FIP's rule set
atomically (single nft transaction on the NAT node), then the old IRIP is released. *Why:*
simplest lifecycle with no orphan tracking; the IRIP is invisible externally, so churn is
harmless. Consequence: IRIP pool sizing = max concurrently *bound* FIPs (+ headroom).

### D9 — Announcement responsibilities
Path A: `ovn-network-agent` announces the FIP /32 toward upstream (R§3). Path B: NAT nodes
announce the FIP — the **aggregate pool prefix** where a pool is exclusively `external-nat`
(guaranteed by D6), otherwise per-/32 — and the `ovn-network-agent` announces the IRIP /32
into the DC fabric (X§6.4–6.6). No component ever announces an address it does not realize.

### D10 — Brownfield pods must import before enabling
Enabling the IPAM driver on a pod with pre-existing FIPs without importing them into the
master would let the master hand out already-used addresses to other pods. The import
procedure (§9) is therefore a **mandatory** step in the enablement runbook, and `eab-manage
import` is a deliverable (M3).

### D11 — Pool scoping is enforced in the master
Each public `ManagedSubnet` carries `access ∈ {public, dedicated}`; `dedicated` pools own a
set of scope entries (`SubnetScope`: project, domain, or CloudPod — multiple entries
OR-combine). The master's reserve path filters candidate pools by scope and rejects
out-of-scope requests — including specific-IP requests — with **403**. An honest error is
chosen over masking; that this reveals the existence of a reserved range is accepted for an
operator-run platform. *Why:* Neutron RBAC (`access_as_external`) shapes what tenants see,
but a mis-configured RBAC rule or network mapping must not hand out customer addresses —
scope enforcement is central, like I1 (⇒ I6). Consequences: `NetworkMapping` resolution
becomes 1:N (one Neutron network may map to several managed subnets, §9.4 Model 2);
`project_id` turns from informational into authorization-relevant and is mandatory for
dedicated pools; pool creation validates CIDR *overlap* against all managed subnets (a
unique CIDR is not enough once customers bring prefixes); and `POST /v1/admin/mappings`
rejects mappings that would attach a dedicated pool to a CloudPod outside its `cloudpod`
scope.

### D12 — Deterministic pool selection
For a reserve without a specific IP or subnet: an explicitly requested `subnet_id` always
wins; otherwise the master picks from *eligible* pools — dedicated pools (in scope for the
requesting project/domain) before `public` pools, tie-broken by pool id. *Why:* a customer
with a dedicated range expects to land in it by default, and the fixed order makes
allocation behavior reproducible. Consequence: a scoped customer who wants a generic public
IP must request the public subnet explicitly — documented UX trade-off (§9.4).

### D13 — BYOIP/BYOAS pools are Path B only, announced from one NAT cluster
Pools with `prefix_origin = customer` or a non-NULL `origin_asn` must have
`realization = external-nat`; the master rejects other combinations (validation analogous
to D6). The pool aggregate is announced from exactly **one** NAT cluster, pinned via
`nat_cluster`. Two announcement modes: **B1** (default) — the operator ASN originates the
customer prefix and the customer's RPKI ROA authorizes the operator ASN (the AWS BYOIP
model); **B2** (opt-in, BYOAS) — the NAT nodes originate with the customer ASN as BGP
origin (per-pool route-map `set as-path prepend <customer-asn>`, or a per-customer
VRF/`local-as` instance), the ROA authorizes the customer ASN, and the upstream must
explicitly accept the path (IRR/filters, §9.5). *Why:* Path A would replicate per-prefix
origin policy onto every network node with per-/32 churn; Path B centralizes BGP policy on
few NAT nodes and already announces aggregates for exclusive pools (D6/D9). Consequence:
the exporter contract gains pool metadata (`GET /v1/nat/pools`, §6.3), and BYO onboarding
becomes a gated runbook step (§9.5, I7).

### D14 — Domain resolution in the agent, fail closed
Scope checks need the requesting project's domain, which Neutron's IPAM path does not
carry. The agent resolves `project_id → domain_id` via its existing read-only Keystone
credentials (R§5.2 `[neutron]`) with a TTL cache and passes `domain_id` in the master
reserve payload. If Keystone is unreachable and the candidate set contains domain-scoped
pools, the reserve fails 503 (fail closed, in the spirit of D1). *Why:* the IPAM driver
must stay lightweight inside neutron-server, and the agent already holds Keystone
credentials. Consequence: Keystone becomes a soft dependency of the allocation path — for
domain-scoped pools only, cushioned by the cache, and irrelevant for project-/pod-scoped or
public pools.

## 6. Interface Contracts

### 6.1 API surface and versioning

All master and agent endpoints live under `/v1`. Within `v1`, changes are **additive only**
(new endpoints, new optional fields); breaking changes require `/v2`. The FastAPI apps
generate OpenAPI documents; CI publishes them as artifacts, and the contract tests (§10) run
the real clients against a live master.

Consolidated endpoint inventory (existing = R§5.3/R§6.5, X = X§6.2, new = this document):

| Endpoint | Source | Milestone |
|---|---|---|
| `POST /v1/addresses/reserve` | R§6.5 | M1 |
| `POST /v1/addresses/release` | R§6.5 | M1 |
| `POST /v1/addresses/confirm` | **new (D7)** | M1 |
| `GET /v1/addresses` | R§6.5 | M1 |
| `POST /v1/cloudpods/heartbeat`, `GET /v1/cloudpods` | R§6.5 | M1 |
| `POST /v1/admin/{subnets,cloudpods,mappings}` + `GET` variants | R§6.5 | M1 |
| `POST /v1/admin/addresses/force-release` | **new (R§12.2 item)** | M1 |
| `POST /v1/admin/subnets/{id}/scopes` + `GET`/`DELETE` variants | **new (D11)** | M1 |
| `GET /v1/health` (master and agent) | R§5.3/R§6.5 | M1/M2 |
| Agent: `POST /v1/addresses/{reserve,release}` | R§5.3 | M2 |
| Agent: `POST /v1/nat/{bind,unbind}` (local, from L3 driver) | X§6.3 | M4 |
| `POST /v1/nat/bind`, `POST /v1/nat/unbind`, `GET /v1/nat/mappings` | X§6.2 | M4 |
| `GET /v1/nat/pools` | **new (D13)** | M4 |

Error model (uniform): `400` invalid input (e.g. IP outside managed pool), `401`
unauthenticated, `403` token/CloudPod mismatch or pool-scope violation (D11), `404` unknown
mapping/allocation, `409` conflict (already allocated / pool exhausted), `503` upstream
authority unreachable.

### 6.2 Reserve/confirm additions (D7, D11, D14)

`reserve` request gains an optional `idempotency_key` (string, unique per reservation, D7)
and an optional `domain_id` (resolved and supplied by the agent, D14; used by the master
only for scope filtering, D11); response is unchanged. Scope violations — including
specific-IP requests into an out-of-scope dedicated pool — return `403`. New endpoint:

```
POST /v1/addresses/confirm
{ "ip_address": "198.51.100.42", "network_id": "<neutron-net-uuid>" }
→ 200 { "ip_address": ..., "state": "active" }   (idempotent; 404 if unknown)
```

The acting CloudPod is derived from the token, as everywhere (R§11.3).

### 6.3 Exporter → NAT controller contract (D5)

The exporter delivers **desired state**, not events:

- Incremental push on every bind/unbind: the full record
  `{fip, irip, state, version}` for the affected FIP.
- Periodic full document: the complete active mapping list (equivalent to
  `GET /v1/nat/mappings`).
- Pool metadata (`GET /v1/nat/pools`, D13): one record per `external-nat` pool —
  `{cidr, origin_asn, announce_mode, nat_cluster, authz_state}`. The NAT controller derives
  its FRR origination (aggregate + origin ASN) from this document, never from mapping
  records; pools with `authz_state ≠ verified` are omitted (I7).

The NAT controller must be idempotent: applying the same record twice is a no-op; applying a
record replaces the FIP's rules in one atomic nft transaction (D8); a FIP absent from a full
document gets its rules removed. The controller reports rule application back (ack), which
transitions `nat_state: pending → active` (X§6.2).

## 7. Consolidated Data Model

Base: R§6.3 (`CloudPod`, `ManagedSubnet`, `NetworkMapping`, `Allocation`) plus X§6.2
extensions, refined as follows:

```
ManagedSubnet            (R§6.3, extended)
  + kind          : 'public' | 'internal'        # 'internal' = IRIP pool (replaces the
                                                 #  config-only irip_pool_cidr from X§9)
  + realization   : 'ovn' | 'external-nat'       # public pools only; enforces D6
  + access        : 'public' | 'dedicated'       # D11 (public pools only)
  + owner         : string, nullable             # customer reference, informational (D11)
  + prefix_origin : 'operator' | 'customer'      # D13 (BYOIP)
  + origin_asn    : int, nullable                # D13 (BYOAS; NULL = operator ASN)
  + nat_cluster   : string, nullable             # D13 (Path B aggregate pinning)
  + authz_state   : 'pending' | 'verified'       # I7 (customer prefixes start pending)

SubnetScope              (new, D11)
  managed_subnet_id : FK → ManagedSubnet
  scope_type        : 'project' | 'domain' | 'cloudpod'
  scope_value       : string
  # UNIQUE(managed_subnet_id, scope_type, scope_value); rows OR-combine

Allocation               (R§6.3 + X§6.2, extended)
  + state             : 'pending' | 'active'     # D7
  + lease_expires_at  : datetime, nullable       # D7 (set while pending)
  + idempotency_key   : string, nullable, unique # D7
  + target_ip / target_port / nat_state / nat_node   # X§6.2 (Path B only)

InternalAddress          (X§6.2, refined)
  - pool_cidr (string)  → replaced by managed_subnet_id FK to a ManagedSubnet
                          with kind='internal'   # unifies pool logic with _find_free_ip
```

Rationale for the `kind` unification: IRIP allocation then reuses the exact allocation
machinery (pool iteration, unique constraint, race retry, R§8.3) instead of a parallel
implementation.

Two reserve-path consequences of D11: the `(cloudpod_id, neutron_network_id)` mapping
lookup returns a *list* of managed subnets (§9.4 Model 2 puts several pools behind one
shared network), and `POST /v1/admin/subnets` validates CIDR **overlap** against all
existing managed subnets including IRIP pools — the unique constraint on `cidr` only
rejects identical strings, which is insufficient once customers bring their own prefixes.

Schema management: Alembic migrations from the first migration onward;
`Base.metadata.create_all` remains a dev-only bootstrap (as already noted in R§6.6).

## 8. Packaging and Deployment Reference

- **One Python distribution** with extras, as specified in R§4.2: the IPAM/L3 drivers stay
  lightweight (imported into neutron-server); `[agent]` / `[master]` pull service deps.
- **Artifacts:** pip package; container images `eab-master`, `eab-agent`,
  `eab-nat-controller`; systemd units + sample configs under `etc/` (R§4.1).
- **Reference topology:** master ≥2 replicas behind a LB on HA PostgreSQL (D2); one agent per
  controller node bound to localhost (R§5); ≥2 NAT nodes with ECMP/anycast and identical rule
  sets (X§6.4).
- **Ecosystem integration:** OSISM/kolla deployment roles are an optional M5 deliverable; the
  design must not depend on any specific deployment tool.

## 9. Brownfield Adoption and Pod Lifecycle

### 9.1 Enabling EAB on a pod with existing FIPs (D10)

1. Register the pod, managed subnets, and network mappings at the master (R§9.2).
2. Enter a FIP-change freeze (maintenance window) for the pod.
3. Run `eab-manage import --cloudpod <id>`: lists all FIPs on managed external networks from
   local Neutron and registers each via `reserve` with the specific IP, created directly as
   `state=active`. **Conflicts (already allocated to another pod) are the duplicates the
   whole system exists to find** — the tool reports them; the operator resolves manually
   before proceeding.
4. Enable `ipam_driver = external_address_broker`, start the agent, restart neutron-server
   (R§9.3).
5. Verify: local FIP count equals the master's allocation count for this pod; the first sync
   reports no discrepancies; lift the freeze.

### 9.2 Provisioning pools for both paths (D6)

Operators using Path B create separate public managed subnets per realization path (e.g.
`198.51.100.0/24` → `ovn`, `203.0.113.0/24` → `external-nat`) plus at least one internal pool
(`kind=internal`) for IRIPs, sized per D8. Each pod maps its corresponding Neutron networks
onto them (R§9.2). The per-pod disjoint allocation pools for router gateways (R§8.6) remain
required until "broker all allocations" lands (M5).

### 9.3 Decommissioning a pod

Extends R§8.5: mark the pod offline, confirm BGP withdrawal of all its /32s (and that no
`external-nat` mappings still target IRIPs in that pod), then force-release its allocations
via the admin endpoint (M1) — never automatically.

### 9.4 Provisioning dedicated pools (D11, D12)

Two Neutron-side models; both map onto a `dedicated` ManagedSubnet with scope entries:

1. **Dedicated external network per customer (recommended).** In every CloudPod within the
   pool's scope, create a separate external network + subnet for the customer range, make
   it visible via Neutron RBAC (`openstack network rbac create --action access_as_external
   --target-project ...`), and map it onto the dedicated ManagedSubnet. The range's
   existence is not leaked to other tenants, and specific-IP requests from other networks
   already fail the pool-membership check (400).
2. **Additional subnet on the shared external network.** No network proliferation, but
   every tenant of the shared network can see the subnet (Neutron cannot hide subnets per
   project) and scope enforcement rests entirely on the master (403, I6). Requires the 1:N
   mapping resolution (D11). Use when a network per customer is a concern.

In both models the R§8.6 rule applies: local (non-FIP) allocation pools must not overlap
dedicated ranges. In Model 1 the customer's own gateway ports consume the customer range —
correct, but size the pool for it; in Model 2 keep dedicated subnets out of the shared
network's local allocation pools entirely.

Scopes are **admin-managed** (`/v1/admin/subnets/{id}/scopes`, `eab-manage`); customer
self-service is future evolution (§12).

### 9.5 BYOIP / BYOAS onboarding (D13, I7)

Per customer prefix, before the pool may become `authz_state = verified`:

1. **Authorization evidence:** LOA from the prefix holder; IRR `route:` object for the
   prefix with the intended origin ASN; RPKI ROA — authorizing the **operator ASN** (mode
   B1, default) or the **customer ASN** (mode B2).
2. **Upstream coordination:** prefix filters and (B2) as-path filters updated; max-prefix
   budgets checked. B2 must be explicitly accepted by the upstream — an operator-AS path
   ending in a foreign origin is otherwise indistinguishable from origin spoofing.
3. **Master provisioning:** create the ManagedSubnet (`kind=public`,
   `realization=external-nat`, `prefix_origin=customer`, `access=dedicated`, scope entries,
   `origin_asn` for B2, `nat_cluster`); CIDR-overlap validation runs here (D11).
4. **Verification:** looking-glass check — aggregate visible with the expected origin ASN,
   RPKI state `valid` — then set `authz_state=verified`. Before that, the exporter never
   ships the pool (I7).

**Offboarding** reverses the order: drain/release all allocations, withdraw the aggregate
(remove the pool from the export), remove upstream filters and ROA/IRR entries, delete the
pool.

## 10. Test and Verification Strategy

| Level | Setup | What it proves |
|---|---|---|
| **Unit** (tox, every PR) | none | Factory routing logic (device_owner / external-network checks, R§4.6); allocation race retry; lease reaper; IRIP allocator; exporter idempotency. |
| **Contract** (every PR) | master + Postgres in containers | Real `AgentClient`/`MasterClient` against a live master; OpenAPI conformance; error-model table (§6.1). |
| **Functional race storm** (every PR) | master + Postgres + 2 simulated agents (docker-compose) | **I1**: N parallel reserves from 2 pods ⇒ zero duplicates; pool exhausts at exactly pool-size allocations; lease expiry releases unconfirmed reservations. |
| **Scope enforcement** (every PR) | master + Postgres in containers | **I6**: out-of-scope reserve and specific-IP requests ⇒ 403; selection order per D12; CIDR-overlap rejection at pool creation; domain-resolution failure with domain-scoped candidates ⇒ fail closed (D14). |
| **Single-pod integration** (gate) | one devstack/kolla-AIO with driver enabled | FIP CRUD via broker; specific-IP requests; induced port-create failure (quota) ⇒ reservation auto-released within lease TTL; agent down ⇒ FIP create fails 503 (**I3**). |
| **Multi-pod e2e** (periodic) | 2 AIO pods + 1 master | Concurrent FIP create/delete loops across pods ⇒ **I1** holds; kill master mid-storm ⇒ both pods fail closed, existing FIPs keep passing traffic (**I5**), no duplicates after recovery; brownfield import scenario (§9.1) incl. seeded duplicate detection. |
| **Path B data path** (lab, then periodic) | X§8 Phase 0 lab: NAT nodes + FRR + one pod | `ovn-nbctl lr-nat-list` shows no public FIP for `external-nat` routers; inbound/outbound flows per X§7; ECMP asymmetry (two NAT nodes, directions forced onto different nodes — must work statelessly); drift injection (delete an nft rule ⇒ reconcile restores it within `reconcile_interval`); NAT node failure ⇒ flows unaffected; BYO pool aggregate carries the expected origin ASN (B1/B2) and is announced only from the pinned NAT cluster (D13). |
| **Failure drills** | staging | Walk R§8.1–8.6 as an operator checklist per release. |

## 11. Implementation Roadmap

Milestones are strictly ordered by dependency, except S1/S2 which run in parallel to M1–M3.
The allocation core (M1–M3) delivers standalone value — conflict-free distributed floating
IPs with today's OVN realization — even if Path B were never built.

### M0 — Bootstrap and feasibility spikes
- Repo skeleton per R§4.1; tox, lint, unit-test CI; Alembic scaffolding.
- **Spike S1 (gates M4 only):** minimal L3 flavor passthrough driver on the target release
  (2024.x/2025.x): verify router/interface/gateway operations can be re-delegated to the OVN
  backend while FIP operations are intercepted (X§6.1 caveat). Exit: written go/no-go with
  the concrete delegation mechanism, or a fallback plan (e.g. carrying a small Neutron patch
  upstream).
- **Spike S2 (optional, parallel):** X§8 Phase 0 — NAT data path standalone (nftables + FRR,
  manual mapping), proving stateless 1:1 + ECMP before any OpenStack coupling.

*Acceptance:* CI green on the empty skeleton; S1 report merged.

### M1 — EAB master
- Data model §7 (incl. D7 columns, `kind`/`realization`, the D11/D13 pool-tier columns and
  `SubnetScope`); Alembic migration 001.
- API per §6.1 (M1 rows): reserve (with idempotency), release, **confirm**, list, heartbeat,
  admin CRUD (incl. **scope CRUD**, D11), **admin force-release**.
- Scoped reserve path (D11/D12): 1:N mapping resolution, eligibility filter + 403,
  deterministic pool selection, CIDR-overlap validation at pool creation, mapping
  validation against `cloudpod` scopes.
- Background tasks: lease reaper (D7), CloudPod offline marking
  (`cloudpod_offline_timeout`, R§12.2).
- Prometheus metrics endpoint; audit log of every reserve/confirm/release with acting pod.

*Acceptance:* race-storm suite green (I1); scope-enforcement suite green (I6); reaper test
green; OpenAPI artifact published.

### M2 — Agent + IPAM driver (single pod, Path A complete)
- Agent per R§5 (app, master client, cache, sync incl. pending-confirmation logic per D7).
- IPAM driver + `AddressRequestFactory` per R§4; `AFTER_DELETE` callback (R§4.7) plus the new
  `AFTER_CREATE` confirm callback (D7); callback import wiring per R§4.8.
- Verify idempotency-key availability (FIP port ID at `get_request` time) — adjust D7 note.
- Agent-side domain resolution (D14): `project_id → domain_id` via the existing read-only
  Keystone credentials, TTL cache, fail closed; `domain_id` added to the master payload.

*Acceptance:* single-pod integration suite green (incl. fail-closed and leak-cleanup tests,
§10); a FIP allocated via broker is realized in OVN and announced by `ovn-network-agent`
exactly as before (Path A regression-free).

### M3 — Multi-pod hardening and operations
- Multi-pod e2e environment and chaos suite (§10).
- `eab-manage` CLI: `import` (§9.1), `force-release`, `report` (allocation/discrepancy
  overview).
- Brownfield runbook (§9.1) validated against a pod with pre-existing FIPs.
- `eab-manage import` respects pool scopes (imported FIPs violating a scope are reported as
  conflicts); dedicated-pool provisioning runbook (§9.4) validated.
- Rate limiting on the master; dashboards for the R§10.1 metrics.

*Acceptance:* multi-pod chaos suite green incl. master-outage scenario (I3/I5); import of a
pod with seeded duplicates produces the expected conflict report.

### M4 — External NAT path (Path B) — *gated by S1*
- Master: NAT extensions (§7 Path B columns, `InternalAddress`, `/v1/nat/*`), IRIP allocator
  reusing pool machinery, exporter + reconcile (D5, §6.3), D6 validation (`bind` rejected for
  `ovn` pools).
- Neutron: `external-nat` L3 flavor driver with delegation per S1; Option C NAT row
  (`external_ip = IRIP`, D4); agent `bind_nat`/`unbind_nat` passthrough (X§6.3).
- NAT controller: consumes desired state, programs nftables (notrack, raw priority — X§6.4)
  + FRR; aggregate announcement for exclusive pools (D9).
- `ovn-network-agent` (separate repo, coordinated release): IRIP /32 announcement into the
  fabric, keyed off NAT rows whose `external_ip` is an IRIP (X§6.5–6.6).
- Pool metadata export (`GET /v1/nat/pools`) + NAT-controller FRR origination templating
  per pool: origin-ASN modes B1/B2, aggregate pinned to `nat_cluster` (D13); BYOIP/BYOAS
  onboarding runbook (§9.5) validated in the lab, incl. the I7 gating.

*Acceptance:* X§7 flows demonstrated end-to-end on the lab; Path B suite (§10) green incl.
ECMP asymmetry, drift repair, and no-public-FIP-in-OVN check; FIP re-association swaps IRIP
without traffic to other FIPs being touched (D8); a BYO pool is announced with its
configured origin ASN and RPKI-validates in the lab (D13).

### M5 — Scale-out and evolution
- HA validation (master replicas + Patroni failover drill, D2).
- IPv6: allocation is address-family-agnostic already; Path B realizes IPv6 routing-only (no
  NAT, X§8 Phase 3).
- Quota management per pod/project (R§12.3); webhook notifications (R§12.3).
- "Broker all allocations on managed external networks" — removes the R§8.6 partitioning.
- Optional: OSISM/kolla deployment integration; web dashboard.

*Acceptance:* per item; defined when scheduled.

### Traceability

| Vision element (§1) | Decisions | Delivered by |
|---|---|---|
| Any IP in any pod, exactly once | D1, D2, D7 | M1–M3 |
| Routing follows allocation | D9 | M2 (Path A), M4 (Path B) |
| Realization per router | D3, D4, D6 | M4 (gated by S1) |
| IP as pure control-plane object | D5, D8 | M4 |
| Moves without renumbering | (§12, out of scope) | future |
| "Private" public addresses + BYOIP/BYOAS (§4.3) | D11–D14 | M1–M3 (scoping), M4 (BYO) |

## 12. Future Evolution (explicitly out of scope, must not be precluded)

- **Cross-pod FIP mobility:** re-associating a FIP to a workload in *another* pod without
  renumbering. Path B makes this a mapping update (new IRIP in the target pod) plus a
  Neutron-side ownership transfer — the natural payoff of the mapping-service model. Requires
  a master-side "allocation transfer" operation; nothing in the current design blocks it.
- **Overlapping tenant CIDRs without IRIPs:** per-tenant encapsulation / EVPN Type-5 at the
  NAT layer (X§5, X§10).
- **Customer self-service for dedicated pools:** tenants managing their own pool scopes and
  BYO onboarding status behind a customer-facing auth model (Keystone). Deliberately
  admin-managed until then (D11, §9.4).
- **Keystone-based authentication** between agents and master (R§11.2, R§12.3).
- **Hierarchical / multi-region brokers** if a single master domain becomes too large.

## 13. Consolidated Risk Register

| # | Risk | Impact | Mitigation | Tracked in |
|---|---|---|---|---|
| 1 | L3 flavor framework skips flavored routers entirely; delegation to OVN may not be cleanly possible on the target release | Path B blocked | Spike S1 before any M4 investment; fallback: upstream patch or postponing Path B (core unaffected) | M0 |
| 2 | Master/DB outage blocks all FIP creation platform-wide | Ops incident | D2 (replicas + Patroni), monitoring R§10.1, I5 keeps data plane up | M1, M5 |
| 3 | Reservation leaks from failed Neutron transactions | Pool erosion | D7 leases + reaper (deterministic), sync grace as backup | M1/M2 |
| 4 | Brownfield enablement without import creates instant duplicates | Routing conflict | D10 mandatory runbook + `eab-manage import` conflict report | M3 |
| 5 | Mixed pools break aggregate announcement | Blackholed FIPs | D6 pool property, master-side validation | M4 |
| 6 | Outbound traffic of IRIP sources bypasses the NAT layer | Asymmetric/broken egress | Fabric policy routing + explicit test in Path B suite (X§10) | M4 |
| 7 | Dual source of truth (Neutron / master / NAT layer) drifts | Stale rules | I4 + reconcile with drift-injection tests | M4 |
| 8 | Stateless NAT ⇒ no shared N:1 SNAT for FIP-less VMs behind `external-nat` routers | Feature gap | Documented constraint (X§10); separate stateful egress service if needed | docs |
| 9 | Security-group/conntrack asymmetry with stateless NAT + ECMP | Dropped flows | Explicit ECMP-asymmetry test; SGs enforced at the VM port on the IRIP side (X§10) | M4 |
| 10 | Non-FIP ports (router gateways) collide across pods | Routing conflict | R§8.6 disjoint-pool operating rule until M5 "broker all allocations" | M3 docs, M5 |
| 11 | Mis-created mapping or RBAC rule exposes a dedicated pool to foreign tenants | Customer-IP leakage | I6 master-side scope enforcement; mapping creation validated against `cloudpod` scopes (D11) | M1 |
| 12 | Keystone dependency in the allocation path (domain-scoped pools) | FIP-create latency/outage | TTL cache; fail closed only when domain-scoped pools are candidates (D14) | M2 |
| 13 | Upstream rejects customer-origin announcements (B2) | BYOAS pool unusable | B1 (operator origin) as default; B2 opt-in with §9.5 coordination checklist | M4, docs |
| 14 | BYOIP prefix overlaps an existing managed pool or IRIP pool | Broken routing/allocation | CIDR-overlap validation at pool creation (D11) | M1 |
| 15 | Pool aggregate announced from a NAT cluster without the pool's mappings | Blackholed FIPs | `nat_cluster` pinning; announcement follows pool metadata only (D13) | M4 |
