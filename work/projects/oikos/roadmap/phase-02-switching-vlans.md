---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-2, switching, vlan, network]
confidence: high
---

# Phase 2 — Managed Switching & VLAN Rollout

## Objective

Replace any dumb/consumer switches with a **managed L2+/L3 switch**, then roll out the VLAN plan: at minimum mgmt (10), servers (20), IoT (60), camera (70), guest (80), home (100). VLAN 30 (k8s), 40 (storage), 50 (ai) get provisioned in their own phases but the trunks are ready.

## Why now

You have a firewall (Phase 1) but everything is still on one flat LAN. VLANs are the second half of a real network boundary. Also, you can't build Wi-Fi (Phase 4) or run internal services (Phase 3) without VLANs already carrying the trunk to each room.

## Prerequisites

- Phase 1 complete (OPNsense running on the WAN edge)
- Baseline understanding of 802.1Q, trunk vs access ports (study track Labs 02 + 03)
- Maintenance window (household DHCP will move VLAN — brief downtime)

## Hardware

| Item | Required? | Buy timing | Candidates | Rough cost |
|---|---|---|---|---|
| **Managed L2+/L3 switch**, 24+ ports, 802.1Q, PoE++ (at least 8 PoE for APs) | **Required** | BUY NOW | MikroTik CRS328-24P-4S+RM · Ubiquiti USW-Pro-24-PoE · Cisco Catalyst 1200/1300 series · TP-Link TL-SG3428MP | US$300–700 |
| Cat6 patch cables (≥ 6) | Required | BUY NOW | any brand | US$25 |
| Cat6 stapled / structured drop cables | Deferred to Phase 4 | WAIT | — | — |
| Small rack shelf / mount | Optional | BUY SOON | any 1U shelf | US$25 |

**CCNA-value note**: If cost allows, pick a **Cisco Catalyst 1200/1300 series** — the CLI is closest to what you'll see on the CCNA exam. MikroTik and Ubiquiti are cheaper and more feature-rich but use different CLIs / UIs. See ADR to record the choice.

## Software

- Switch firmware — latest stable at time of purchase
- OPNsense: additional VLAN interfaces + rules
- (Nothing to install on Proxmox yet)

## Network changes

- New physical switch inserted between OPNsense and the household
- Trunk between OPNsense (or OPNsense LAN port) and switch — carries VLANs 10, 20, 60, 70, 80, 100
- Each VLAN gets its own OPNsense interface + firewall rules + DHCP scope
- Existing family devices remain on VLAN 100 (home) — invisible change to them
- T14 Proxmox migrates from ISP DHCP to `172.27.10.11/24` on VLAN 10 (mgmt)

## Implementation sequence

1. **Prototype in Packet Tracer** — study track Labs 02 (VLANs), 03 (trunks), 04 (router-on-a-stick / inter-VLAN routing). Do not skip these.
2. **Buy + unbox** — take a firmware inventory, note the CLI/UI you'll be using.
3. **Bench-configure the switch** — not in the household network yet:
   - Set mgmt IP `172.27.10.2/24` (or vendor-specific)
   - Create VLANs 10, 20, 60, 70, 80, 100
   - Configure the port that will trunk to OPNsense: trunk, native VLAN 100 (or a dedicated native), tagged 10/20/60/70/80
   - Configure access ports per VLAN (label them physically with tape)
   - Save config
4. **Configure OPNsense VLANs**:
   - Interfaces → Assignments → add VLAN 10 (parent = LAN NIC), VLAN 20, VLAN 60, VLAN 70, VLAN 80
   - Each: IP `172.27.<vlan>.1/24`, DHCP scope `.100–.199`
   - Firewall rules per VLAN (see policy matrix below)
5. **Cutover** (maintenance window):
   - Power switch off, unrack ISP-swap dumb switch (if any)
   - Rack managed switch, connect OPNsense trunk to designated trunk port
   - Connect family clients to VLAN 100 access ports
   - Verify family DHCP + Internet from a family device before leaving
6. **Move T14 to mgmt VLAN**:
   - Configure a switch port as access VLAN 10
   - Plug T14 into it
   - On T14, edit `/etc/network/interfaces` to static `172.27.10.11/24` with gateway `172.27.10.1`
   - `ifreload -a`; verify SSH from workstation (now on VLAN 100) — will fail without an inter-VLAN rule; add rule for your workstation-IP → mgmt-IP over HTTPS/SSH
7. **Firewall rules baseline** — see policy matrix below.
8. **Break/fix drill** (below).
9. **Document** — runbooks, diagrams, current-state.

## VLAN firewall policy matrix (baseline)

| From ↓ To → | mgmt | servers | iot | camera | guest | home |
|---|---|---|---|---|---|---|
| mgmt | — | any | any | any | any | any |
| servers | — | — | deny | deny | deny | deny |
| iot | deny | deny | — | deny | deny | deny |
| camera | deny | deny | deny | — | deny | deny |
| guest | deny | deny | deny | deny | — | deny |
| home | your workstation only | published ports only | any (control IoT) | NVR ports | — | — |

Everything → WAN: allowed by default except camera (deny to Internet).

## CCNA topics reinforced

- 802.1Q tagging, native VLAN
- Access vs. trunk ports
- VLAN pruning / allowed VLAN list
- Inter-VLAN routing (via OPNsense = "router on a stick")
- ACL direction and placement
- Broadcast domains and MAC-table isolation
- STP basics — one switch means no loops yet, but you'll see `spanning-tree` in `show run`

## Study track lab (Packet Tracer)

- **Lab 02**: two VLANs (10, 20) on one switch; two PCs each; ping succeeds within VLAN, fails across
- **Lab 03**: add trunk to a second switch; verify VLAN membership traverses
- **Lab 04**: add router with subinterfaces (`encapsulation dot1Q 10`); enable inter-VLAN routing; ACL to block VLAN 20 → 10

## Physical Oikos lab exercise

- On the switch: `show vlan brief`, `show interfaces trunk`, `show mac address-table`
- On a client in VLAN 60 (iot): try to ping something in VLAN 10 (mgmt) — must fail
- On your workstation in VLAN 100: try to reach the T14 mgmt UI — must succeed only via the explicit inter-VLAN rule
- Change a switch port from access VLAN 100 to VLAN 60, plug a laptop in — laptop's IP should change subnets

## Break/fix drill

1. Misconfigure the OPNsense-to-switch trunk — set the wrong native VLAN on one side. Observe: family clients lose Internet (or partial connectivity). Diagnose from switch CLI (`show interfaces trunk`) and OPNsense logs.
2. Recover.
3. Delete an inter-VLAN firewall rule you rely on. Observe: mgmt UI unreachable from your normal client. Recover via console.

## Verification commands

Switch (Cisco-style — adapt for your vendor):

```
show vlan brief
show interfaces trunk
show interfaces status
show mac address-table
show running-config
```

OPNsense:

```bash
configctl interface list
pfctl -sr | grep -i <vlan>
tcpdump -i vlan10 host <mgmt-target>
```

Client:

```bash
ip -br a         # confirm subnet by VLAN
ping <gateway>
ping <cross-vlan-target>   # must fail per policy
traceroute <cross-vlan-target>
```

## Security posture at this phase

- Mgmt UIs (switch, OPNsense, Proxmox): reachable **only from mgmt VLAN + your workstation IP**
- Camera VLAN: deny Internet egress (rule at OPNsense)
- Guest VLAN: deny all RFC1918 destinations
- IoT VLAN: deny to mgmt/servers; allow Internet
- Log all denies for a week; then dial down noisy rules

## Monitoring coverage

- Switch SNMP enabled (v3), community string in password manager
- OPNsense already logging; add per-VLAN traffic graphs
- Still no Prometheus (Phase 5)

## Kephaleos docs to create / update

- **Create**: `runbooks/switch-baseline.md` (vendor-specific config)
- **Create**: `runbooks/opnsense-vlan-rollout.md`
- **Create**: `runbooks/inter-vlan-firewall-policy.md` (the matrix + rationale)
- **Update**: `current-state.md`, `inventory/nodes.md`, `architecture/address-plan.md`
- **Update**: `nodes/pve-t14-01.md` — IP now static `172.27.10.11`
- **Append**: `CHANGELOG.md`
- **New ADR**: `adr/000N-switch-vendor.md` (record why you picked the vendor)

## Diagram updates

- Master architecture: add managed switch + trunk + VLAN labels
- New: VLAN diagram (logical) — one Mermaid page in `diagrams/`
- Update: current-state diagram

## Completion checklist

- [ ] Switch installed and configured with all target VLANs
- [ ] Trunk to OPNsense working; native VLAN choice recorded
- [ ] VLAN interfaces + DHCP scopes live on OPNsense
- [ ] Firewall policy matrix implemented and tested against
- [ ] T14 on mgmt VLAN at `172.27.10.11/24`
- [ ] Family clients on VLAN 100 without noticing anything
- [ ] Break/fix drill completed
- [ ] Runbooks, diagrams, docs updated
