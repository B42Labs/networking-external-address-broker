# External NAT Realization — Placing Floating IPs Outside of OVN

> **Status:** Design / Draft
> **Scope:** Extension to the existing EAB architecture (see [README.md](README.md)).
> Describes how a floating IP is **allocated via Neutron** but the actual **NAT is
> realized outside of OpenStack/OVN** — analogous to AWS Elastic IPs.

---

## 1. Context and Goal

### 1.1 What EAB solves today

The External Address Broker (EAB) decouples floating IP **allocation** from the individual
CloudPod: a central master is the source of truth, and the Neutron IPAM driver asks the local
agent before every FIP allocation, so the same public IP is never assigned in two CloudPods
at once. The **realization** of the FIP still happens classically in OVN today (a
`dnat_and_snat` NAT entry), and the `ovn-network-agent` announces the corresponding /32 route
via BGP.

### 1.2 What this document adds

The goal is the **third decoupling layer**: the FIP should exist purely as a *mapping record*,
and the 1:1 NAT should be applied by a layer **outside of OVN**. For such a FIP, **no NAT
entry** should be created in OVN.

| Layer | EAB today | This document |
|---|---|---|
| **Allocation** (who owns the IP) | ✅ Neutron IPAM → EAB master | unchanged |
| **Association** (FIP ↔ VM) | ✅ Neutron `floatingip` | unchanged (API stays) |
| **Realization / NAT** (packet translation) | OVN `dnat_and_snat` | **external, not in OVN** |

### 1.3 The AWS model

In AWS the Elastic IP exists nowhere in the instance's data path — the instance only has its
private VPC IP. The EIP is an entry in the mapping service (EIP → private IP), and the
**stateless 1:1 NAT is applied at the edge of the AWS network** (the Internet Gateway layer).
We replicate exactly this separation on OVN.

---

## 2. Terminology

| Term | Meaning |
|---|---|
| **FIP** | Floating IP — the public, internet-routed address. Allocated by Neutron/EAB. |
| **IRIP** | *Internal Routable IP* — an address of the VM that is **unique and internally routable** within the operator's routing domain. Equivalent to the "ENI private IP" in AWS. |
| **NAT layer** | External component (outside OpenStack) that statelessly 1:1-translates `FIP ↔ IRIP` and announces the FIP /32 to the internet. Equivalent to the AWS Internet Gateway. |
| **Mapping record** | The record `FIP → IRIP` (+ tenant/port context). Lives in the EAB master. Equivalent to the AWS mapping-service entry. |

---

## 3. Current State: How OVN Realizes a FIP Today

Full analysis in the OVN docs. Short version — a FIP is in OVN **exactly one row** in the
northbound `NAT` table on the logical router:

```
type          : dnat_and_snat
external_ip   : "198.51.100.42"          # floating IP
logical_ip    : "10.0.0.5"               # fixed IP of the VM
logical_port  : "<neutron-port-uuid>"    # the VM's LSP (distributed)
external_mac  : "fa:16:3e:94:8e:89"      # distributed only
gateway_port  : <uuid lrp-...>
external_ids  : {"neutron:fip_id"=..., "neutron:fip_external_mac"=..., ...}
```

From this, `ovn-northd` derives the logical flows (`lr_in_dnat`, `lr_out_snat`, ARP
responder), `ovn-controller` renders OpenFlow on the responsible chassis, and the
`ovn-network-agent` reads exactly these NAT entries to announce the /32 via BGP.

**The row is written by** `OVNClient._create_or_update_floatingip`
(`neutron/plugins/ml2/drivers/ovn/mech_driver/ovsdb/ovn_client.py`), invoked by the OVN L3
service plugin.

> **Key takeaway:** "FIP not present in OVN" = **this `dnat_and_snat` row must not be
> created**. That is precisely the technical lever.

---

## 4. Target Architecture

```
  ┌─ Control plane — allocation + mapping ───────────────────
  │
  │   Neutron floatingip API
  │     ├─ (a) IPAM ──────────► EAB IPAM driver → agent → master
  │     │                       (conflict-free FIP pool)
  │     └─ (b) L3 realization ─► L3 flavor driver "external-nat"
  │                • writes NO OVN NAT entry
  │                • reports FIP ↔ IRIP mapping to the master
  └──────────────────────────────────────────────────────────
                  │
                  │  mapping export (webhook / API)
                  ▼
  ┌─ Data plane — external NAT layer (outside OpenStack) ─────
  │
  │   nftables / VPP / DPU
  │     • stateless 1:1   FIP ↔ IRIP
  │     • BGP announce    FIP/32 ──────────► internet
  └──────────────────────────────────────────────────────────
                  │
                  │  dest = IRIP (the public IP is gone)
                  ▼
  ┌─ DC fabric / OVN — routes only the IRIP ─────────────────
  │
  │   routes ONLY the IRIP to the VM
  │     • IRIP unique + announced via BGP (ovn-network-agent)
  │     • OVN: pure L3 routing — NO NAT, NO public IP
  └──────────────────────────────────────────────────────────
```

The public FIP thus appears in **only** two places: in the Neutron/EAB record (bookkeeping)
and in the external NAT layer (data path). In OVN it is invisible.

---

## 5. The Central Design Decision: Delivering the Translated Packet

The NAT itself (`FIP → IRIP`) is trivial. The actual problem is the same one AWS solves with a
mapping service + encapsulation: **after the NAT layer has translated to the internal address
— how does the packet reach the right VM?** Tenant fixed IPs may overlap (`10.0.0.5` in many
tenants).

We choose the **"unique internal routable IP" (IRIP) variant** — the simplest option that
avoids the overlap problem and fits EAB's already centrally managed address space:

- Every VM with a FIP gets an IRIP that is **unique within the operator's routing domain**
  (no overlap on the path NAT layer ↔ OVN).
- The IRIP is announced via BGP **internally** (not to the internet) from the VM's compute
  node (`ovn-network-agent`/`ovn-bgp-agent`), so the NAT layer can reach any VM by its IRIP.
- **OVN routes only the IRIP** — plain L3 routing, no NAT, no public IP.

> **Constraint:** IRIPs must be unique on the path NAT layer ↔ OVN. EAB manages an **IRIP
> pool** for this in addition to the FIP pool (see §6.2).

**Future option (overlapping tenant CIDRs):** instead of unique IRIPs, the NAT layer
encapsulates with a per-tenant VNI (becomes an OVN chassis / EVPN Type-5). More powerful but
significantly more complex, and deliberately **out of scope** here (see §10).

---

## 6. Components and Concrete Work Items

### 6.1 Neutron: L3 flavor `external-nat` + driver

**Mechanism.** Since release 2023.2, the OVN L3 functionality (routers *and* floating IPs) has
been refactored into a swappable driver. Via the **flavor / service-provider framework**, a
custom driver can be loaded per router to handle realization.

**Approach — "passthrough driver that overrides only FIP operations":**

The driver delegates router/interface/gateway realization to the normal OVN logic (so OVN
keeps doing internal routing), but **overrides the floating IP events**:
- `create_floatingip` / `update_floatingip`: **no** `dnat_and_snat` in OVN; instead report the
  `FIP↔IRIP` mapping to the EAB master.
- `delete_floatingip`: withdraw the mapping at the master / NAT layer.

Configuration (`neutron.conf`):

```ini
[service_providers]
service_provider = L3_ROUTER_NAT:external-nat:networking_external_address_broker.l3.driver.ExternalNatL3Driver
```

Create the flavor and a router using it:

```bash
openstack network flavor create external-nat --service-type L3_ROUTER_NAT
openstack network flavor profile create --driver \
    networking_external_address_broker.l3.driver.ExternalNatL3Driver
openstack network flavor add-profile external-nat <profile-id>

openstack router create --flavor external-nat r-public
```

Driver skeleton (verify module paths against your Neutron release):

```python
# networking_external_address_broker/l3/driver.py
from neutron.services.l3_router.service_providers import base
from neutron_lib.callbacks import events, registry, resources

from networking_external_address_broker.client import agent_client


class ExternalNatL3Driver(base.L3ServiceProvider):
    """L3 flavor driver: FIP NAT is NOT realized in OVN but exported to the
    external NAT layer. Routers/interfaces are still realized by OVN.
    """

    def __init__(self, l3_plugin):
        super().__init__(l3_plugin)
        self._client = agent_client.AgentClient()
        registry.subscribe(self._fip_created,
                           resources.FLOATING_IP, events.AFTER_CREATE)
        registry.subscribe(self._fip_updated,
                           resources.FLOATING_IP, events.AFTER_UPDATE)
        registry.subscribe(self._fip_deleted,
                           resources.FLOATING_IP, events.AFTER_DELETE)

    # Important: NO call into OVN NAT realization.
    # The flavor dispatcher ensures the OVN driver does not process
    # these FIPs.

    def _fip_created(self, resource, event, trigger, payload):
        fip = payload.latest_state
        if not fip.get('port_id'):      # only realize associated FIPs
            return
        self._client.bind_nat(
            floating_ip=fip['floating_ip_address'],
            fixed_ip=fip['fixed_ip_address'],
            port_id=fip['port_id'],
            fip_id=fip['id'],
        )

    def _fip_updated(self, resource, event, trigger, payload):
        old = payload.states[0]
        new = payload.latest_state
        if old.get('port_id') and not new.get('port_id'):
            self._client.unbind_nat(floating_ip=new['floating_ip_address'])
        elif new.get('port_id'):
            self._client.bind_nat(
                floating_ip=new['floating_ip_address'],
                fixed_ip=new['fixed_ip_address'],
                port_id=new['port_id'],
                fip_id=new['id'],
            )

    def _fip_deleted(self, resource, event, trigger, payload):
        fip = payload.latest_state
        self._client.unbind_nat(floating_ip=fip['floating_ip_address'])
```

**Work items:**
- [ ] Implement the driver module `l3/driver.py` (override FIP events, delegate routers to OVN).
- [ ] `setup.cfg`: register the entry point / service provider.
- [ ] Verify that **no** `dnat_and_snat` row is created in OVN NB for FIPs on `external-nat` routers (`ovn-nbctl lr-nat-list`).
- [ ] Check the release-specific base class / module paths against the target version (e.g. 2024.1/2024.2).

### 6.2 EAB master: extend data model + API

The existing `Allocation` model is extended with the mapping part (FIP → target IRIP + NAT
state), and an **IRIP pool** is added (unique internal addresses, cf. §5).

```python
# Additions in eab_master/db/models.py

class Allocation(Base):
    # ... existing fields ...
    # realization / mapping part:
    target_ip   = Column(String(64), nullable=True)   # IRIP of the VM
    target_port = Column(String(36), nullable=True)   # Neutron port UUID
    nat_state   = Column(String(16), default='unbound')  # unbound|pending|active|error
    nat_node    = Column(String(64), nullable=True)   # which NAT instance holds the rule


class InternalAddress(Base):
    """Unique internal routable IP (IRIP) per VM port."""
    __tablename__ = 'internal_addresses'
    id        = Column(Integer, primary_key=True, autoincrement=True)
    ip        = Column(String(64), nullable=False, unique=True)  # globally unique
    cloudpod_id = Column(String(64), ForeignKey('cloudpods.id'), nullable=False)
    port_id   = Column(String(36), nullable=True)
    pool_cidr = Column(String(64), nullable=False)
```

New master endpoints:

```
POST /v1/nat/bind     {floating_ip, fixed_ip, port_id, fip_id, cloudpod_id}
                      -> assigns/creates IRIP, sets nat_state=pending,
                         triggers export to NAT layer, returns {target_ip}
POST /v1/nat/unbind   {floating_ip, cloudpod_id}  -> remove rule
GET  /v1/nat/mappings -> all active FIP↔IRIP mappings (for reconcile/export)
```

**Work items:**
- [ ] Extend `Allocation` with `target_ip`/`target_port`/`nat_state`/`nat_node` (Alembic migration).
- [ ] Add `InternalAddress` table + IRIP pool allocation (analogous to `_find_free_ip`).
- [ ] Implement `/v1/nat/bind` + `/v1/nat/unbind` + `/v1/nat/mappings`.
- [ ] Idempotency + reconcile (mapping list as source of truth for the NAT layer).

### 6.3 EAB agent: export to the NAT layer

The agent receives `bind_nat`/`unbind_nat` from the L3 driver (locally) and forwards them to
the master; the master (or a dedicated exporter) pushes the rule to the NAT layer. There are
two ways to connect the NAT layer — a **push API** to the NAT controller, or **BGP-driven**
(the NAT layer pulls the /32 + mapping itself). Recommendation: push API with periodic full
reconcile against `/v1/nat/mappings`.

**Work items:**
- [ ] `AgentClient.bind_nat/unbind_nat` (local path L3 driver → agent).
- [ ] Master-side exporter (webhook/gRPC/Netconf depending on the NAT layer) — cf. README §12.3 "Webhook notifications".
- [ ] Reconcile loop: mapping list ↔ actual state of the NAT layer.

### 6.4 External NAT layer (reference implementation)

Stateless 1:1 NAT + /32 announce. Start with Linux + nftables + FRR (scaled horizontally via
ECMP), optionally VPP/DPU later.

Example (per mapping `FIP ↔ IRIP`):

```bash
# 1:1 NAT (stateless, no conntrack needed)
nft add rule ip nat PREROUTING  ip daddr 198.51.100.42 dnat to 10.128.0.5
nft add rule ip nat POSTROUTING ip saddr 10.128.0.5    snat to 198.51.100.42

# Announce the FIP /32 to the internet (FRR)
vtysh -c 'configure terminal' \
      -c 'router bgp 65000' \
      -c 'address-family ipv4 unicast' \
      -c 'network 198.51.100.42/32'
```

Properties: stateless ⇒ any NAT instance can hold any rule ⇒ trivially redundant via
anycast/ECMP. The NAT layer routes the IRIP sources toward the internet (SNAT), and the DC
fabric routes inbound IRIP destinations to the VM (via internal BGP, §6.5).

**Work items:**
- [ ] NAT controller that consumes `/v1/nat/mappings` and programs nftables + FRR.
- [ ] HA: ≥2 NAT nodes, ECMP/anycast, identical rule set (stateless ⇒ no state sync).
- [ ] Ensure the outbound default route of the IRIP sources goes through the NAT layer.

### 6.5 Internal routability of the IRIP (`ovn-network-agent`)

So that inbound (DNAT'd) packets to the IRIP reach the VM, the IRIP is announced **internally**
via BGP from the VM's compute node — analogous to today's FIP /32 logic, but with the
**internal** address and into the **DC fabric** (not the internet).

**Work items:**
- [ ] Announce the VM's IRIP as a /32 internally on the compute host (extension/variant of the existing `ovn-network-agent`).
- [ ] Ensure OVN delivers the IRIP to the VM (pure routing, no NAT).

### 6.6 `ovn-network-agent`: adjust behavior

Since **no** OVN NAT entries exist anymore for `external-nat` FIPs, the existing
`ovn-network-agent` will not see these FIPs — which is correct: the **FIP /32 is now announced by
the NAT layer**, no longer by OVN. For the internal path, the agent announces the **IRIP**
(§6.5).

**Work items:**
- [ ] Disable public-FIP announcement for `external-nat` routers (no OVN NAT source present anymore — usually automatic).
- [ ] Add the internal IRIP announcement.

---

## 7. End-to-End Flows

### 7.1 Create and associate a FIP

```
User → Neutron: floating ip create + set --port
  1. IPAM: EAB driver → agent → master  (FIP from pool, conflict-free)   [existing]
  2. floatingips DB record                                               [existing]
  3. L3 flavor driver "external-nat":
       - NO ovn dnat_and_snat
       - bind_nat(FIP, fixed_ip, port) → master
  4. Master: assign IRIP, nat_state=pending, export to NAT layer
  5. NAT layer: 1:1 rule FIP↔IRIP, FIP/32 BGP → internet; nat_state=active
  6. ovn-network-agent: announce IRIP/32 internally
```

### 7.2 Inbound packet (internet → VM)

```
Internet → FIP 198.51.100.42
  → NAT layer (holds FIP/32):  DNAT  198.51.100.42 → IRIP 10.128.0.5
  → DC fabric routes 10.128.0.5 (internally via BGP) → VM's compute node
  → OVN: pure L3 routing to the VM port (NO NAT, FIP never visible)
```

### 7.3 Outbound packet (VM → internet)

```
VM (source IRIP 10.128.0.5) → default route → NAT layer
  → SNAT  10.128.0.5 → 198.51.100.42
  → internet (source = FIP)
```

### 7.4 Remove a FIP

```
delete/disassociate → L3 driver: unbind_nat → master: nat_state=unbound,
  export removal → NAT layer: rule + /32 announce gone;
  IPAM release (FIP) as before (EAB AFTER_DELETE callback).
```

### 7.5 Failover

- **NAT layer:** stateless ⇒ ECMP/anycast, loss of a node is non-critical (no state).
- **VM's compute:** IRIP /32 announcement follows the VM (like today's FIP /32 logic).

---

## 8. Implementation Plan (Phases)

### Phase 0 — Validate the NAT layer (no OVN changes)
- [ ] Build the NAT layer (nftables+FRR) + NAT controller standalone.
- [ ] Manually set one `FIP↔IRIP` mapping, verify inbound/outbound + BGP.
- [ ] **Still with** OVN FIP in parallel — only to prove the NAT layer's data path.

### Phase 1 — Mapping in the EAB master
- [ ] Data model (`Allocation` extension, `InternalAddress`, IRIP pool).
- [ ] `/v1/nat/bind|unbind|mappings` + exporter to the NAT layer + reconcile.

### Phase 2 — Suppress OVN realization (the actual core)
- [ ] L3 flavor `external-nat` + passthrough driver (§6.1).
- [ ] Verify: **no** `dnat_and_snat` row for these FIPs.
- [ ] `ovn-network-agent` IRIP announcement (§6.5/§6.6).

### Phase 3 — Hardening
- [ ] HA for the NAT layer (anycast/ECMP), reconcile robustness.
- [ ] Orphan handling (NAT rule without a Neutron FIP and vice versa).
- [ ] Metrics (active mappings, NAT node health, reconcile drift).
- [ ] IPv6 path (1:1 is trivial; possibly no NAT at all, pure routing).

---

## 9. Configuration (Examples)

`neutron.conf`:
```ini
[DEFAULT]
ipam_driver = external_address_broker          # existing (allocation)

[service_providers]
service_provider = L3_ROUTER_NAT:external-nat:networking_external_address_broker.l3.driver.ExternalNatL3Driver
```

`eab-master.conf` (addition):
```ini
[nat]
# IRIP pool (internally unique, NOT routed to the internet)
irip_pool_cidr = 10.128.0.0/16
# NAT layer connection
exporter = webhook
exporter_endpoint = https://nat-controller.internal:8443/v1/rules
reconcile_interval = 30
```

---

## 10. Risks, Trade-offs, Open Questions

| Topic | Assessment |
|---|---|
| **IRIP uniqueness** | Variant A requires non-overlapping internal addresses on the path NAT↔OVN. EAB manages the IRIP pool centrally. Overlapping tenant CIDRs ⇒ future option encap/EVPN (out of scope). |
| **Dual source of truth** | FIP↔IRIP exists in the Neutron DB *and* EAB master *and* NAT layer. Reconcile (`/v1/nat/mappings`) makes the master authoritative. |
| **L3 flavor maturity** | Flavor/driver refactor since 2023.2; test the FIP path against the concrete target version (module paths, event signatures). |
| **Outbound steering** | The default route of the IRIP sources must reliably traverse the NAT layer (fabric policy/routing). |
| **Security groups / conntrack** | Stateless 1:1 NAT has no own state; security groups remain effective in OVN at the VM port (IRIP side). Check inbound/outbound asymmetry. |
| **MTU** | No extra header with pure NAT (no overlay overhead) — an advantage over the encap variant. |

---

## 11. What Changes in Existing EAB — and What Does Not

**Unchanged:**
- IPAM driver, agent, master allocation logic, allocation conflict-freedom across CloudPods.
- The Neutron `floatingip` API for end users.

**New / changed:**
- New **L3 flavor driver** (`external-nat`) — suppresses OVN NAT, exports the mapping.
- **Master data model** extended with mapping + IRIP pool; new `/v1/nat/*` endpoints.
- **External NAT layer** as a new data-path component (outside OpenStack).
- **`ovn-network-agent`**: no longer announces a public /32 for these FIPs, instead the internal IRIP.

> **Result:** EAB becomes, from a pure *address bookkeeper*, a *mapping service* in the AWS
> sense; the L3 flavor driver becomes the counterpart of the *distributed Internet Gateway*;
> and the FIP exists exactly where it does in AWS: as a record in the control plane and as a
> 1:1 NAT rule at the network edge — **not** in the overlay/OVN.
