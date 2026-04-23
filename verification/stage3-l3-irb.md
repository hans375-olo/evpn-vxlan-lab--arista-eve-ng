# Stage 3 Verification — L3 EVPN (Symmetric IRB)

## Objective
Hosts in different VLANs/subnets can communicate, with routing performed on the
local leaf. Anycast gateway provides local default gateway on both leaves
simultaneously. Type 5 IP prefix routes advertised per VRF.

---

## LEAF-1

### show bgp evpn route-type ip-prefix
```
LEAF-1#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65001

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.3:1 ip-prefix 10.10.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.4:1 ip-prefix 10.10.10.0/24
                                 10.0.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.0.0.4:1 ip-prefix 10.10.10.0/24
                                 10.0.0.4              -       100     0       65000 65002 i
 * >      RD: 10.0.0.3:1 ip-prefix 10.10.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.4:1 ip-prefix 10.10.20.0/24
                                 10.0.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.0.0.4:1 ip-prefix 10.10.20.0/24
                                 10.0.0.4              -       100     0       65000 65002 i
```

**Result:**
- Both leaves advertising both subnets as Type 5 routes — anycast gateway working
  as designed; both leaves are simultaneously active gateways for both subnets
- LEAF-2 routes (RD 10.0.0.4:1) received via both spines with ECMP (`>Ec` / `ec`)
- Locally originated routes show next-hop `-` (directly connected via SVI)

### show ip route vrf TENANT-A
```
LEAF-1#sh ip route vrf TENANT-A

VRF: TENANT-A

Gateway of last resort is not set

 C        10.10.10.0/24 is directly connected, Vlan10
 C        10.10.20.0/24 is directly connected, Vlan20
```

**Result:** Both tenant subnets directly connected via SVIs within VRF TENANT-A.
Inter-VLAN routing active at the leaf level — no spine involvement required for
routed traffic.

### show vxlan vni
```
LEAF-1#sh vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet3       untagged
                                    Vxlan1          10
10020       20         static       Vxlan1          20

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF            Source
----------- ---------- -------------- ------------
99999       4097       TENANT-A       evpn
```

**Result:**
- VNI 10010 → VLAN 10, access port Ethernet3 (HOST-A) plus VXLAN tunnel
- VNI 10020 → VLAN 20, VXLAN tunnel only (HOST-C on LEAF-2)
- VNI 99999 → dynamic VLAN 4097, VRF TENANT-A, source `evpn` — L3 VNI for
  symmetric IRB confirmed active. VLAN 4097 is an internal system VLAN
  assigned automatically by vEOS; not user-configured.

---

## Notes on symmetric IRB

In symmetric IRB, the ingress leaf (where the source host is connected) performs
both the routing decision and VXLAN encapsulation. The egress leaf decapsulates
and delivers locally. Traffic always traverses the L3 VNI (99999) in both
directions — hence "symmetric".

This contrasts with asymmetric IRB where the ingress leaf routes and the egress
leaf bridges, requiring every leaf to hold every VLAN. Symmetric IRB is the
production-standard approach: leaves only need to know their own VLANs, and the
L3 VNI carries inter-subnet traffic across the fabric.

---

## Host reachability test

HOST-A (10.10.10.1, VLAN 10, LEAF-1) → HOST-C (10.10.20.1, VLAN 20, LEAF-2):
ping successful. Inter-VLAN routing via symmetric IRB confirmed.

Traffic path:
1. HOST-A sends packet to anycast gateway 10.10.10.254 (local SVI on LEAF-1)
2. LEAF-1 routes within VRF TENANT-A, encapsulates in VXLAN with L3 VNI 99999
3. Packet traverses fabric to LEAF-2 VTEP (10.0.0.4)
4. LEAF-2 decapsulates, routes within VRF TENANT-A, delivers to HOST-C on VLAN 20

---

## vEOS 4.30 deprecation note

Configuring `rd` under `vrf instance` produces the following warning on vEOS 4.30:

```
Since the RD is required for BGP operation, please configure the RD for VRF
TENANT-A under the 'router bgp vrf' submode. The configuration of the RD under
the VRF definition submode is deprecated and no longer required.
```

Configs updated: `vrf instance TENANT-A` contains no RD statement. RD is
configured solely under `router bgp / vrf TENANT-A`. See `lessons-learned.md`.
