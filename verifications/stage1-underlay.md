# Stage 1 Verification — Underlay (eBGP)

## Objective
All loopbacks reachable across the fabric. Both leaves reachable via two equal-cost
paths through SPINE-1 and SPINE-2.

---

## SPINE-1

### show ip bgp summary
```
SPINE-1#sh ip bgp summ
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  LEAF-1                   10.1.1.1 4 65001             96        96    0    0 01:18:49 Estab   1      1
  LEAF-2                   10.1.1.3 4 65002             87        90    0    0 01:10:48 Estab   1      1
```

### show ip route
```
SPINE-1#sh ip rou

VRF: default

Gateway of last resort is not set

 C        10.0.0.1/32 is directly connected, Loopback0
 B E      10.0.0.3/32 [200/0] via 10.1.1.1, Ethernet1
 B E      10.0.0.4/32 [200/0] via 10.1.1.3, Ethernet2
 C        10.1.1.0/31 is directly connected, Ethernet1
 C        10.1.1.2/31 is directly connected, Ethernet2
```

**Result:** Both leaf loopbacks (10.0.0.3, 10.0.0.4) present as `B E` routes via
correct interfaces. Sessions Established.

---

## SPINE-2

### show ip bgp summary
```
SPINE-2#sh ip bgp summ
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  LEAF-1                   10.1.2.1 4 65001             67        65    0    0 01:09:xx Estab   1      1
  LEAF-2                   10.1.2.3 4 65002             67        67    0    0 01:09:xx Estab   1      1
```

### show ip route
```
SPINE-2#sh ip rou

VRF: default

Gateway of last resort is not set

 C        10.0.0.2/32 is directly connected, Loopback0
 B E      10.0.0.3/32 [200/0] via 10.1.2.1, Ethernet1
 B E      10.0.0.4/32 [200/0] via 10.1.2.3, Ethernet2
 C        10.1.2.0/31 is directly connected, Ethernet1
 C        10.1.2.2/31 is directly connected, Ethernet2
```

**Result:** Identical to SPINE-1. Both leaf loopbacks reachable, sessions Established.

---

## LEAF-1

### show ip route (underlay, pre-overlay)
```
 C        10.0.0.3/32 is directly connected, Loopback0
 B E      10.0.0.1/32 [200/0] via 10.1.1.0, Ethernet1
 B E      10.0.0.2/32 [200/0] via 10.1.2.0, Ethernet2
 B E      10.0.0.4/32 [200/0] via 10.1.1.0, Ethernet1
                               via 10.1.2.0, Ethernet2
```

**Result:** LEAF-2 loopback (10.0.0.4) reachable via two equal-cost paths —
ECMP across both spines confirmed. `maximum-paths 2` working as intended.

---

## Notes
- Spine-to-spine ping not expected to work — no direct links between spines and no
  transit path exists in this topology. This is correct behaviour for a pure
  eBGP spine-leaf fabric (RFC 7938). Spines are not required to reach each other;
  they are route reflectors for the leaf tier only.
- `PfxRcd 1` on each spine session at this stage = each leaf advertising its own
  loopback only. Correct.
