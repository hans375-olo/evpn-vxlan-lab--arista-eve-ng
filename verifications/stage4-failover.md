# Stage 4 Verification — Failure Testing

## Objective
Demonstrate fabric resilience under single spine link failure. Document convergence
behaviour, traffic impact, and restoration.

## Test setup
- Continuous ping HOST-A (10.10.10.1) → HOST-C (10.10.20.1) at 200ms intervals
- Failures introduced by shutting Ethernet1 (SPINE-1 link) then Ethernet2
  (SPINE-2 link) on LEAF-1 sequentially
- Show commands captured immediately after each shutdown

---

## Test 1 — LEAF-1 Ethernet1 shutdown (SPINE-1 link)

### show bgp evpn summary (immediately after shutdown)
```
LEAF-1#sh bgp evpn summ
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 65001
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  SPINE-1                  10.1.1.0 4 65000            126       135    0    0 00:00:20 Idle(NoIf)
  SPINE-2                  10.1.2.0 4 65000            127       138    0    0 01:28:05 Estab   7      7
```

**Observation:** SPINE-1 session drops to `Idle(NoIf)` immediately — physical link
loss detected without waiting for BGP hold timer. SPINE-2 session unaffected,
`PfxRcd 7` maintained.

### show ip route vrf TENANT-A (immediately after shutdown)
```
 C        10.10.10.0/24 is directly connected, Vlan10
 B E      10.10.20.1/32 [200/0] via VTEP 10.0.0.4 VNI 99999 router-mac 50:00:00:cb:38:c2 local-interface Vxlan1
 C        10.10.20.0/24 is directly connected, Vlan20
```

**Observation:** Routing table unchanged. VTEP reachability to 10.0.0.4 maintained
via surviving SPINE-2 path. No route withdrawal, no traffic interruption.

### Ping results
- 1-2 packets observed at 50-60ms latency (vs baseline <20ms)
- Zero packet drops

### show bgp evpn summary (after no shutdown)
```
  SPINE-1                  10.1.1.0 4 65000            139       154    0    0 00:00:08 Estab   7      7
  SPINE-2                  10.1.2.0 4 65000            128       140    0    0 01:29:26 Estab   7      7
```

**Observation:** SPINE-1 session re-established in ~8 seconds, full prefix table
restored. Both sessions Established, `PfxRcd 7` on both.

---

## Test 2 — LEAF-1 Ethernet2 shutdown (SPINE-2 link)

### show bgp evpn summary (immediately after shutdown)
```
  SPINE-1                  10.1.1.0 4 65000            139       156    0    0 00:00:30 Estab   7      7
  SPINE-2                  10.1.2.0 4 65000            128       140    0    0 00:00:06 Idle(NoIf)
```

**Observation:** Identical behaviour — physical link loss detected immediately,
SPINE-1 session unaffected, routing maintained via surviving path.

### show ip route vrf TENANT-A
Identical to Test 1 — routing table unaffected throughout.

### BGP notification on restore
```
Apr 22 14:53:34 LEAF-1 Bgp: %BGP-3-NOTIFICATION: sent to neighbor 10.1.2.0
(VRF default AS 65000) 6/7 (Cease/connection collision resolution) 0 bytes
```

**Observation:** On link restore, both ends attempted to open the BGP session
simultaneously. vEOS resolved the collision by closing one connection (Cease 6/7
= connection collision resolution). Normal behaviour — session came up cleanly
seconds later.

---

## Summary

| Event | Detection method | Reconvergence | Packet loss |
|-------|-----------------|---------------|-------------|
| Spine link failure | Physical signal (immediate) | Sub-second | 0 |
| Spine link restore | Physical signal | ~8 seconds | 0 |

**Key takeaway:** In this topology, link failures are detected via physical signal
rather than BGP hold timer expiry. This is the best-case failure scenario — in
production, if a failure is only detectable via BGP timers (keepalive 3s, hold 9s),
convergence would be slower but the dual-spine ECMP design would still prevent
packet loss once reconvergence completes.

The zero-drop result demonstrates the core value of the dual-spine design: no
single spine link failure can black-hole traffic as long as the surviving path
is installed via ECMP before the failure occurs.
