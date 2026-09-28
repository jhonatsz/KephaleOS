---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-6, eve-ng, gns3, ccna, lab]
confidence: high
---

# Phase 6 — Advanced CCNA Lab (EVE-NG / GNS3)

## Objective

Stand up **EVE-NG Community Edition** (or GNS3) on Proxmox to run real Cisco IOS images. Enables OSPF, RSTP, EtherChannel, IPv6, advanced ACLs, and multi-router topologies that Packet Tracer can't accurately simulate.

## Why now

Packet Tracer has carried the study track through Phases 0–5. Passing CCNA requires realistic behavior of Cisco IOS features Packet Tracer approximates poorly (OSPF neighbor states, STP root election under load, EtherChannel bundle behavior). Phase 6 is when your existing compute + network is stable enough to reliably run a lab environment that needs nested virtualization + several GB of RAM per virtual router.

## Prerequisites

- Phases 0–5 complete
- T14 with headroom: at least 16 GB RAM free after existing VMs (OPNsense-VM interim if applicable, DNS, obs01)
- Nested virtualization enabled on Proxmox (`kvm-intel nested=1` or AMD equivalent)
- IOU/IOS/CSR image acquisition (licensing note below)

## Hardware

| Item | Required? | Buy timing | Notes |
|---|---|---|---|
| Additional RAM for T14 | Conditional | BUY as needed | T14 Gen 2 SODIMM max is often 64 GB; check spec sheet. If nested labs OOM, upgrade. |
| USB-serial console cable | Already own from Phase 1 | — | Optional for console-server realism |
| — | — | — | Everything else runs virtually |

## Software

- **EVE-NG Community Edition** (or **GNS3 server** + Windows/macOS client) — deployed as a VM on Proxmox
- **Cisco IOL/IOS images** — legal path: CCNA study bundles via Cisco NetAcad **or** licensed subscriptions. **Do not source images from grey channels** — record the licensing decision as an ADR.
- **Alternative open images**: FRRouting, VyOS, Cumulus VX — for OSPF/BGP without licensing issues

## Network changes

- New VM: `eve01.oikos.home.arpa` on VLAN 20, `172.27.20.40`
- Bridged network from EVE-NG to a **dedicated isolated VLAN** (e.g., VLAN 90 — reserved for lab traffic that must not leak into production)
- Firewall: block VLAN 90 ↔ everything else by default; allow narrow ports if you need mgmt-VLAN → EVE-NG web UI

## Implementation sequence

1. **Confirm nested virt** — `egrep -c ' vmx| svm' /proc/cpuinfo` on Proxmox; enable in `/etc/modprobe.d/kvm.conf` if not already.
2. **Create EVE-NG VM** — Ubuntu 22.04 base, 8 vCPU, 16–24 GB RAM, 200 GB disk (thin), CPU type `host` (required for nested).
3. **Install EVE-NG** — follow official install for Community Edition.
4. **Import images** — legal Cisco images (from NetAcad or similar), FRRouting, VyOS, Alpine for endpoints.
5. **Web UI + client** — install EVE-NG native client on your workstation.
6. **First lab**: single-area OSPF, three routers (R1, R2, R3), each with a loopback; verify `show ip ospf neighbor` all `FULL`.
7. **Second lab**: RSTP with three switches, watch root bridge election.
8. **Third lab**: EtherChannel between two switches (LACP), then break one link and observe.
9. **Save lab files** to `~/workspace/personal/homelab/labs/eve-ng/NN-slug/`.
10. **Document**.

## CCNA topics reinforced (now full-realism)

- **OSPF single-area** — router-id, network statements, neighbor states, DR/BDR
- **OSPF multi-area** (advanced but on CCNA scope)
- **RSTP/PVST+** — port roles, states, root election, tuning `spanning-tree portfast`, BPDU guard
- **EtherChannel / LACP** — active/passive, load-balance mode, `show etherchannel summary`
- **Advanced ACLs** — extended, time-based, named
- **NAT variants** — dynamic, static, PAT, twice-NAT
- **IPv6** — SLAAC, DHCPv6, static routes, OSPFv3
- **DHCP relay** across router boundaries
- **Basic troubleshooting** — `debug` commands (used sparingly)

## Study track labs (EVE-NG)

Migrate from Packet Tracer where realism matters:

- **Lab 09**: single-area OSPF, three routers
- **Lab 10**: RSTP + EtherChannel
- **Lab 11**: IPv6 dual-stack
- **Lab 12**: multi-area OSPF with ABR
- Prep: pull the CCNA Odom lab index and mirror the ones marked "IOS behavior differs from PT"

## Physical Oikos lab exercise

None — Phase 6 is virtual by design (Cisco IOS images can't legally run on generic hardware in this project). The "real" Oikos network is unaffected.

## Break/fix drill

1. In an OSPF lab: form neighbor adjacency, then intentionally mismatch OSPF hello/dead timers on one router. Neighbor breaks. Explain why from `debug ip ospf events` output. Fix.
2. In an RSTP lab: manipulate priorities so the "wrong" switch becomes root; observe path changes. Restore.
3. In an EtherChannel lab: `shutdown` one member interface; observe traffic redistribution; bring back and verify rebalancing.

## Verification commands (inside IOS)

```
show ip ospf neighbor
show ip ospf interface
show ip route ospf
show spanning-tree
show spanning-tree root
show etherchannel summary
show etherchannel port-channel
show interfaces port-channel N
show ipv6 route
show ipv6 ospf neighbor
```

## Security posture at this phase

- EVE-NG UI: reachable only from mgmt VLAN + your workstation
- Lab VLAN (90) fully isolated from production
- No inbound from Internet
- Root account for EVE-NG: strong password, keys where supported

## Monitoring coverage

- `obs01` scrapes EVE-NG host metrics (node_exporter)
- Lab traffic intentionally not monitored — it's a playground

## Kephaleos docs to create / update

- **Create**: `runbooks/eve-ng-baseline.md`
- **Create**: `runbooks/eve-ng-image-import.md`
- **Create**: `study/eve-ng-labs.md` — index (mirror the `study/packet-tracer.md` structure)
- **Update**: `study/README.md` — add EVE-NG track link
- **Update**: `current-state.md`, `inventory/nodes.md`
- **New ADR**: `adr/000N-eve-ng-image-sourcing.md` (legal chain-of-custody for Cisco images)
- **Append**: `CHANGELOG.md`

## Diagram updates

- Add "CCNA Lab (EVE-NG)" node to master diagram — attached to production only for mgmt-plane; data plane isolated
- Update study track diagram

## Completion checklist

- [ ] Nested virtualization confirmed working on Proxmox
- [ ] EVE-NG installed and reachable
- [ ] Legal image source recorded in ADR
- [ ] First OSPF lab neighbors reach FULL
- [ ] RSTP lab shows correct root election
- [ ] EtherChannel bundle survives one-member failure
- [ ] Lab VLAN isolation verified (no leak to production)
- [ ] Study track index updated
