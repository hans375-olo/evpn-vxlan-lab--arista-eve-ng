# evpn-vxlan-lab--arista-eve-ng

A hands-on EVE-NG lab demonstrating a full EVPN/VXLAN data centre fabric using **Arista vEOS** throughout. The lab covers eBGP underlay design, BGP EVPN overlay, L2 stretch, symmetric IRB for inter-VLAN routing, and failure testing — the core building blocks of a modern DC fabric.

> **Purpose:** Portfolio lab built to deepen practical experience with DC fabric design ahead of a data centre network engineer role. All configs are functional and tested end-to-end in EVE-NG on VMware Workstation (Windows host).

---

## Topology

```
              ┌──────────────────────────────────────┐
              │             UNDERLAY                 │
              │                                      │
              │   ┌──────────┐    ┌──────────┐       │
              │   │ SPINE-1  │    │ SPINE-2  │       │
              │   │  vEOS    │    │  vEOS    │       │
              │   │ AS 65000 │    │ AS 65000 │       │
              │   └────┬─────┘    └─────┬────┘       │
              │        │╲              ╱│            │
              │        │ ╲            ╱ │            │
              │        │  ╲          ╱  │            │
              │   ┌────┴───╲────────╱───┴────┐       │
              │   │         ╲      ╱         │       │
              │   │    ┌─────╲────╱─────┐    │       │
              │   │    │      ╲  ╱      │    │       │
              │   │ ┌──┴───┐   ╲╱  ┌───┴──┐  │       │
              │   │ │LEAF-1│   ╱╲  │LEAF-2│  │       │
              │   │ │ vEOS │  ╱  ╲ │ vEOS │  │       │
              │   │ │AS65001 ╱    ╲ AS65002│ │       │
              │   │ └──┬───┘       └───┬──┘  │       │
              │   │    │               │     │       │
              │   │ HOST-A          HOST-B   │       │
              │   │ VLAN 10         VLAN 10  │       │
              │   │ 10.10.10.1   10.10.10.2  │       │
              │   │                          │       │
              │   │              HOST-C      │       │
              │   │              VLAN 20     │       │
              │   │           10.10.20.1     │       │
              └───┴───────────────────────── ┴───────┘
```

| Device   | Platform     | Role                    | AS    | Loopback    | Management      |
|----------|--------------|-------------------------|-------|-------------|-----------------|
| SPINE-1  | Arista vEOS  | Spine / Route Reflector | 65000 | 10.0.0.1/32 | 192.168.100.11  |
| SPINE-2  | Arista vEOS  | Spine / Route Reflector | 65000 | 10.0.0.2/32 | 192.168.100.12  |
| LEAF-1   | Arista vEOS  | Leaf / VTEP             | 65001 | 10.0.0.3/32 | 192.168.100.13  |
| LEAF-2   | Arista vEOS  | Leaf / VTEP             | 65002 | 10.0.0.4/32 | 192.168.100.14  |

### Host addressing

| Host   | VLAN | Subnet        | IP           | Connected to   |
|--------|------|---------------|--------------|----------------|
| HOST-A | 10   | 10.10.10.0/24 | 10.10.10.1   | LEAF-1 Eth3    |
| HOST-B | 10   | 10.10.10.0/24 | 10.10.10.2   | LEAF-2 Eth3    |
| HOST-C | 20   | 10.10.20.0/24 | 10.10.20.1   | LEAF-2 Eth4    |

---

## Design Decisions

### Underlay: eBGP as IGP

Rather than running OSPF or IS-IS as the underlay IGP, this lab uses **eBGP point-to-point between every spine-leaf pair** — each leaf is in its own AS, spines share AS 65000. This is the modern DC fabric approach (RFC 7938), used by hyperscalers and recommended by Arista for large-scale fabrics.

- No IGP to tune or summarise
- Failure domain per link, not per area
- Consistent tooling — BGP does both underlay and overlay

Point-to-point /31 links on all fabric interfaces. Loopbacks serve as VTEP source addresses and BGP update-source.

### Overlay: BGP EVPN

EVPN runs as a **separate BGP address family (L2VPN EVPN)** over the same sessions, with spines acting as route reflectors. This avoids a full-mesh between leaves as the fabric scales.

VXLAN is the data plane encapsulation (UDP port 4789, VNI-based).

### Route Reflector placement on spines

Spines have full topology visibility and are already in the forwarding path. Placing the RR function on spines avoids dedicated RR nodes and matches production practice.

### All-Arista platform choice

The original design called for Juniper vQFX spines with Arista vEOS leaves to demonstrate multi-vendor interoperability. vEOS was substituted for all nodes for practical reasons: lighter image (~2 GB vs ~4 GB RAM per vQFX node), freely available, and the EVPN/VXLAN feature set is equivalent for lab purposes. Juniper vQFX or vJunos-switch can be substituted for the spine tier without changes to the leaf configs — the EVPN control plane is vendor-agnostic at the protocol level.

---

## Lab Stages

### Stage 1 — Underlay (eBGP)

**Goal:** All loopbacks reachable across the fabric. ECMP across both spines confirmed.

Key config elements:
- /31 fabric links (no ARP, no DR/BDR)
- eBGP sessions on fabric interfaces
- Loopback redistribution into BGP
- `maximum-paths 2` for ECMP across both spines
- `service routing protocols model multi-agent` — required on vEOS for EVPN (configured at this stage)

**Verification:**
```
LEAF-1# show ip bgp summary
LEAF-1# show ip route
LEAF-1# ping 10.0.0.4 source loopback0
```

→ [Stage 1 verification](verification/stage1-underlay.md) | [Configs](configs/underlay/)

---

### Stage 2 — L2 EVPN Overlay (MAC/IP, Type 2 routes)

**Goal:** Two hosts in the same VLAN on different leaves can communicate. MAC learning via BGP control plane, not flood-and-learn.

Key config elements:
- VXLAN tunnel interface (Vxlan1) on each leaf
- BGP EVPN address family activated on leaves and spines
- VNI-to-VLAN mapping (VLAN 10 → VNI 10010)
- Ingress replication list for BUM traffic (no multicast required)
- EVPN Type 2 routes (MAC/IP advertisement)

When HOST-A ARPs for HOST-B, LEAF-1 generates an EVPN Type 2 route advertising HOST-B's MAC/IP. LEAF-2 installs this in its MAC table, suppressing the ARP flood. Traffic is encapsulated in VXLAN and forwarded directly between VTEPs.

**Verification:**
```
LEAF-2# show bgp evpn route-type mac-ip
LEAF-2# show vxlan address-table
LEAF-2# show vxlan vtep
HOST-A$ ping 10.10.10.2
```

→ [Stage 2 verification](verification/stage2-l2-evpn.md) | [Configs](configs/overlay-l2/)

---

### Stage 3 — L3 EVPN (Symmetric IRB)

**Goal:** Hosts in different VLANs/subnets can communicate, with routing performed on the local leaf — not the spine.

Key config elements:
- SVI on each leaf as the default gateway per VLAN
- Anycast gateway MAC (same virtual MAC on both leaves — hosts see one gateway regardless of which leaf they're behind)
- L3 VNI per VRF (VNI 99999 → VRF TENANT-A)
- EVPN Type 5 routes for IP prefix advertisement
- Symmetric IRB: ingress leaf routes and encapsulates via L3 VNI, egress leaf decapsulates and routes locally

**Why symmetric IRB (not asymmetric)?**
Asymmetric IRB requires every leaf to hold every VLAN — doesn't scale. Symmetric IRB uses a dedicated L3 VNI per VRF so leaves only need to know their own VLANs. This is the production-standard approach.

**Verification:**
```
LEAF-1# show bgp evpn route-type ip-prefix
LEAF-1# show ip route vrf TENANT-A
LEAF-1# show vxlan vni
HOST-A$ ping 10.10.20.1     ← HOST-C in VLAN 20 on LEAF-2
```

→ [Stage 3 verification](verification/stage3-l3-irb.md) | [Configs](configs/overlay-l3/)

---

### Stage 4 — Failure Testing

**Goal:** Demonstrate fabric resilience under single spine link failure. Zero packet loss with dual-spine ECMP design.

Test method: continuous ping HOST-A → HOST-C at 200ms intervals while sequentially shutting Ethernet1 (SPINE-1 link) and Ethernet2 (SPINE-2 link) on LEAF-1.

Results:

| Event | Detection | Reconvergence | Packet loss |
|-------|-----------|---------------|-------------|
| Spine link failure | Physical signal (immediate) | Sub-second | 0 |
| Spine link restore | Physical signal | ~8 seconds | 0 |

Key takeaway: link failures are detected via physical signal rather than BGP hold timer expiry. The surviving spine path is already installed via ECMP before the failure occurs, so traffic continues without interruption.

→ [Stage 4 verification](verification/stage4-failover.md)

---

## Repository Structure

```
evpn-vxlan-lab--arista-eve-ng/
├── README.md                    ← this file
├── topology/
│   └── eve-ng-topology.png      ← EVE-NG screenshot
├── configs/
│   ├── underlay/
│   │   ├── spine1.conf          ← Arista vEOS (EOS format)
│   │   ├── spine2.conf
│   │   ├── leaf1.conf
│   │   └── leaf2.conf
│   ├── overlay-l2/
│   │   ├── spine1.conf
│   │   ├── spine2.conf
│   │   ├── leaf1.conf
│   │   └── leaf2.conf
│   └── overlay-l3/
│       ├── leaf1.conf
│       └── leaf2.conf
├── verification/
│   ├── stage1-underlay.md       ← show command outputs + observations
│   ├── stage2-l2-evpn.md
│   ├── stage3-l3-irb.md
│   └── stage4-failover.md
└── notes/
    └── lessons-learned.md       ← issues hit and how they were resolved
```

---

## EVE-NG Setup

### Environment

EVE-NG Community on Ubuntu VM, hosted on VMware Workstation on Windows. Nested virtualisation is required — enable **"Virtualise Intel VT-x/EPT or AMD-V/RVI"** in the VMware VM settings, otherwise QEMU instances run in pure emulation and are extremely slow.

> **Windows host note:** If VMware Workstation crashes repeatedly during lab bring-up, check whether Virtualisation Based Security (VBS) is active. VBS conflicts with VMware's hypervisor. See `notes/lessons-learned.md` for the resolution.

### Image used

| Image      | Version      | Source                                        |
|------------|--------------|-----------------------------------------------|
| vEOS-lab   | 4.30.10M     | Arista Software Downloads (free account)      |

### vEOS image (qcow2 → EVE-NG)

Arista distributes vEOS-lab as a `.qcow2`, which EVE-NG requires. Import with:

```bash
mkdir -p /opt/unetlab/addons/qemu/veos-4.30.10M
cp vEOS-lab-4.30.10M.qcow2 /opt/unetlab/addons/qemu/veos-4.30.10M/hda.qcow2
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### Resource requirements (VMware host)

| Node    | vCPUs | RAM  |
|---------|-------|------|
| SPINE-1 | 2     | 2 GB |
| SPINE-2 | 2     | 2 GB |
| LEAF-1  | 2     | 2 GB |
| LEAF-2  | 2     | 2 GB |
| **Total** | **8** | **8 GB** |

---

## Key Concepts Demonstrated

| Concept | Stage |
|---------|-------|
| eBGP underlay (RFC 7938) | 1 |
| ECMP across spines | 1 |
| BGP EVPN address family | 2 |
| VXLAN encapsulation / VNI mapping | 2 |
| Type 2 MAC/IP routes | 2 |
| BUM traffic / ingress replication | 2 |
| Anycast gateway | 3 |
| Symmetric IRB | 3 |
| Type 5 IP prefix routes | 3 |
| Dual-spine failure resilience | 4 |

---

## References

- [RFC 7938 — Use of BGP for Routing in Large-Scale Data Centers](https://www.rfc-editor.org/rfc/rfc7938)
- [RFC 8365 — A Network Virtualization Overlay Solution Using EVPN](https://www.rfc-editor.org/rfc/rfc8365)
- [Arista EOS EVPN Configuration Guide](https://www.arista.com/en/um-eos/eos-evpn-overview)
- [Arista Validated Design — EVPN Deployment Guide](https://www.arista.com/assets/data/pdf/Whitepapers/Arista_EVPN_Deployment_Guide.pdf)

---

*Lab built as part of ongoing CCNP-level DC skills development. Feedback welcome via Issues.*
