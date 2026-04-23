# Lessons Learned

Issues encountered during the lab build, with root cause and resolution. Included
here because they're the kind of thing that costs hours if you hit them cold.

---

## Environment — EVE-NG on VMware Workstation (Windows)

This lab runs EVE-NG Community on an Ubuntu VM hosted in VMware Workstation on
Windows. Nested virtualisation is required: enable **"Virtualise Intel VT-x/EPT
or AMD-V/RVI"** in the VM settings (Hardware → Processors). Without it, QEMU
runs in pure emulation and the nodes are unusably slow.

---

## Issue 1 — VMware Workstation crashing on Windows host

**Symptom:** VMware Workstation crashed repeatedly when the EVE-NG lab was
brought up. Crashes were non-deterministic — sometimes on node start, sometimes
after several minutes of operation.

**Root cause:** Conflict between VMware's hypervisor and Windows
Virtualisation Based Security (VBS). On Windows 11 hosts with Credential Guard
or Memory Integrity enabled, VBS holds the hypervisor and prevents VMware from
acquiring it. VMware starts but becomes unstable under load.

**Resolution:** Disable VBS on the Windows host.

Check current status:

```
msinfo32 → System Summary → Virtualization-based security
```

To disable, in an elevated PowerShell session:

```powershell
# Disable Memory Integrity (Core Isolation)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" -Name "Enabled" -Value 0

# Reboot required
Restart-Computer
```

After reboot, confirm VBS is no longer running in msinfo32. VMware ran stably
throughout the rest of the lab once this was resolved.

> **Note:** Disabling Memory Integrity reduces kernel-level exploit protection.
> Acceptable on a dedicated lab machine; review your risk posture on a
> general-purpose machine.

---

## Issue 2 — EVPN sessions showing `Estab(NotNegotiated)`

**Symptom:** After configuring the EVPN address family on both leaves, `show bgp
evpn summary` on the leaves showed:

```
SPINE-1   10.1.1.0  4  65000   Estab(NotNegotiated)
SPINE-2   10.1.2.0  4  65000   Estab(NotNegotiated)
```

No EVPN routes exchanged. VXLAN address table empty.

**Root cause:** Spines had no EVPN address family configured. The BGP session
was up at the IPv4 level, but the EVPN capability was never negotiated — the
spines didn't know what to do with the leaves' EVPN capability advertisement.

**Resolution:** Added `address-family evpn` activation and a permit route-map to
both spines:

```
! SPINE-1
router bgp 65000
   neighbor 10.1.1.1 route-map EVPN-OUT out
   neighbor 10.1.1.3 route-map EVPN-OUT out
   address-family evpn
      neighbor 10.1.1.1 activate
      neighbor 10.1.1.3 activate

route-map EVPN-OUT permit 10
```

```
! SPINE-2
router bgp 65000
   neighbor 10.1.2.1 route-map EVPN-OUT out
   neighbor 10.1.2.3 route-map EVPN-OUT out
   address-family evpn
      neighbor 10.1.2.1 activate
      neighbor 10.1.2.3 activate

route-map EVPN-OUT permit 10
```

Sessions renegotiated within seconds of applying the config. No reboot required.

---

## Issue 3 — RD deprecation warning on vEOS 4.30

**Symptom:** Configuring `rd` under `vrf instance` produced the following
warning on vEOS 4.29 / 4.30:

```
Since the RD is required for BGP operation, please configure the RD for VRF
TENANT-A under the 'router bgp vrf' submode. The configuration of the RD under
the VRF definition submode is deprecated and no longer required.
```

**Root cause:** EOS moved RD configuration out of `vrf instance` and into the
BGP submode. The RD under `vrf instance` is ignored in newer EOS versions —
only the RD under `router bgp / vrf TENANT-A` takes effect.

**Resolution:** Removed `rd` from all `vrf instance` blocks. RD is configured
solely under `router bgp`:

```
router bgp 65001
   vrf TENANT-A
      rd 10.0.0.3:1
      ...
```

---

## Issue 4 — Failure testing results and observations

Continuous ping HOST-A (10.10.10.1) → HOST-C (10.10.20.1) at 200ms intervals
during sequential single spine link failures on LEAF-1.

**Results:**

| Event | Detection method | Reconvergence | Packet loss |
|-------|-----------------|---------------|-------------|
| Spine link failure | Physical signal (immediate) | Sub-second | 0 |
| Spine link restore | Physical signal | ~8 seconds | 0 |

**Observations:**

- Link failure was detected via physical signal — BGP session dropped to
  `Idle(NoIf)` immediately, without waiting for hold timer expiry (which would
  be 9s with default keepalive/hold settings). Best-case failure scenario.
- The surviving spine path was already installed via ECMP before the failure
  occurred. Traffic continued with 1-2 packets at elevated latency (50-60ms vs
  baseline <20ms), zero drops.
- Routing table was unaffected throughout — VTEP reachability maintained via
  the surviving spine.
- On link restore, a BGP collision resolution event fired (`Cease 6/7`) — both
  ends attempted to open the session simultaneously. vEOS closed one and kept
  the other. Normal behaviour, session re-established cleanly seconds later.

**Production note:** In environments where link failure is only detectable via
BGP timers (e.g. a failure mid-path rather than on the direct link), convergence
would be slower (up to hold timer expiry). The dual-spine ECMP design would
still prevent packet loss once reconvergence completes, but there would be a
traffic black-hole window during the hold timer wait. BFD can reduce this to
sub-second in production.

---

## Design note — Platform choice

The original design called for Juniper vQFX spines with Arista vEOS leaves, to
demonstrate multi-vendor interoperability at the EVPN control plane level. vEOS
was substituted for all nodes for practical reasons: the vEOS image is lighter
(~2 GB vs ~4 GB RAM per vQFX node), freely available via Arista's website, and
the EVPN/VXLAN feature set is equivalent for lab purposes.

Juniper vQFX or vJunos-switch can be substituted for the spine tier without
changes to the leaf configs. The EVPN control plane is standardised (RFC 8365)
— the only spine-specific config is the BGP session and EVPN address family
activation, which translates directly to JunOS syntax.
