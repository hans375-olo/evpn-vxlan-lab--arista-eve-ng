# Stage 2 Verification — L2 EVPN Overlay (MAC/IP, Type 2 routes)

## Objective
Two hosts in the same VLAN on different leaves can communicate. MAC learning via
BGP control plane, not flood-and-learn. VXLAN tunnel established between VTEPs.

---

## Troubleshooting note — Estab(NotNegotiated)

On first attempt, EVPN sessions showed the following state on both leaves:

```
SPINE-1   10.1.1.0  4  65000   Estab(NotNegotiated)
SPINE-2   10.1.2.0  4  65000   Estab(NotNegotiated)
```

**Cause:** Spines had no EVPN address family configured. Leaves were advertising
EVPN capability but spines could not respond. The BGP session was up at the IPv4
level but EVPN was never negotiated.

**Fix:** Added `address-family evpn` activation and `route-map EVPN-OUT` to both
spines. Sessions renegotiated within seconds, no reboot required. See
`lessons-learned.md` for full detail.

---

## LEAF-1 — post-fix, hosts connected

### show bgp evpn summary
```
LEAF-1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  SPINE-1                  10.1.1.0 4 65000            129       130    0    0 00:14:59 Estab   2      2
  SPINE-2                  10.1.2.0 4 65000             97       100    0    0 00:14:59 Estab   2      2
```

**Result:** Sessions Established (no NotNegotiated). `PfxRcd 2` = both leaves'
MAC/IP routes being exchanged via spines.

---

## LEAF-2 — hosts connected

### show bgp evpn summary
```
LEAF-2#sh bgp evpn summ
BGP summary information for VRF default
Router identifier 10.0.0.4, local AS number 65002
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  SPINE-1                  10.1.1.2 4 65000             32        35    0    0 00:10:12 Estab   2      2
  SPINE-2                  10.1.2.2 4 65000             31        39    0    0 00:10:00 Estab   2      2
```

### show bgp evpn route-type mac-ip
```
LEAF-2#sh bgp evpn route-type mac
BGP routing table information for VRF default
Router identifier 10.0.0.4, local AS number 65002

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.3:10010 mac-ip 0050.0000.0500
                                 10.0.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.0.0.3:10010 mac-ip 0050.0000.0500
                                 10.0.0.3              -       100     0       65000 65001 i
 * >      RD: 10.0.0.4:10010 mac-ip 0050.0000.0600
                                 -                     -       -       0       i
```

**Result:**
- HOST-A MAC (0050.0000.0500) learned via EVPN, next-hop VTEP 10.0.0.3 (LEAF-1)
- Two equal-cost paths (`>Ec` and `ec`) — one via each spine, ECMP at EVPN layer
- HOST-B MAC (0050.0000.0600) locally originated (`i`)
- MAC was never flooded — learned entirely via BGP control plane (Type 2 route)

### show vxlan address-table
```
LEAF-2#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.0000.0500  EVPN      Vx1  10.0.0.3         1       0:09:57 ago
Total Remote Mac Addresses for this criterion: 1
```

**Result:** HOST-A's MAC installed in VXLAN address table as EVPN type, pointing
to VTEP 10.0.0.3. Not locally learned — installed from BGP control plane.

### show vxlan vtep
```
LEAF-2#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP           Tunnel Type(s)
-------------- --------------
10.0.0.3       unicast, flood
Total number of remote VTEPS:  1
```

**Result:** LEAF-1's VTEP shows both `unicast` and `flood` tunnel types. Unicast
appeared once the first MAC/IP route was received — traffic flows point-to-point
between VTEPs, not via ingress replication for known unicast.

---

## Host reachability test

HOST-A (10.10.10.1, VLAN 10, LEAF-1 Ethernet3) → HOST-B (10.10.10.2, VLAN 10,
LEAF-2 Ethernet3): ping successful. L2 stretch across VXLAN fabric confirmed.
