---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-1, opnsense, firewall, network]
confidence: high
---

# Phase 1 — Production Firewall (OPNsense)

## Objective

Replace the ISP-provided router as the household Internet gateway with **OPNsense** and introduce the first VLAN split (mgmt vs. home). End state: the family still has Internet, plus a real firewall boundary, and management traffic is separated from the family segment.

## Why now

Every subsequent phase (VLAN rollout, DNS, monitoring, VMs) needs a proper firewall boundary. Doing anything sensitive on the ISP-router-flat network is a security anti-pattern. This is also the **highest-household-risk phase**: mess it up and the family Wi-Fi is down until you fix it.

## Prerequisites

- Phase 0 complete (Proxmox running on T14, docs system live)
- ADR oikos-0001 applied (hostname correct)
- Household maintenance window (~2–3 hours; family briefed)
- Existing ISP router credentials accessible

## Hardware

| Item | Required? | Buy timing | Candidates | Rough cost |
|---|---|---|---|---|
| **Firewall appliance** — fanless mini-PC with 4x 2.5 GbE, x86_64, no proprietary NIC | **Required** | BUY NOW | Protectli VP2410 / VP2420; Qotom Q35xxG4; CWWK N100 dual/quad NIC | US$300–500 |
| USB-serial console cable | Recommended | BUY NOW | any USB-A to RJ45 rollover / USB to DB-9 with null modem | US$15 |
| Second short Ethernet patch cable | Required | already own | Cat5e/Cat6 | — |
| USB stick for OPNsense installer | Required | already own | ≥ 4 GB | — |

**Interim fallback (BUY WAIT)**: run OPNsense as a VM on the T14 Proxmox with a USB Ethernet adapter as WAN. Higher risk — the T14 is now a single point of failure for the family Internet. Only do this if the firewall appliance is delayed and the maintenance window has to happen.

## Software

- **OPNsense** — latest stable (25.x line)
- SHA256 verification for the ISO

## Network changes (this is the big one)

- New device: OPNsense sits **between** the ISP ONT/modem and the household network
- ISP modem set to **bridge mode** where possible (else double-NAT, workable but ugly)
- OPNsense WAN: DHCP or PPPoE per ISP
- OPNsense LAN: `172.27.100.1/24` (VLAN 100 — home)
- OPNsense mgmt: `172.27.10.1/24` (VLAN 10 — but this only becomes real once a managed switch exists in Phase 2). For Phase 1: mgmt VLAN lives inside OPNsense but isn't reachable from anywhere yet; the T14 talks to OPNsense via LAN
- Household DHCP served by OPNsense (not the ISP router)
- Family DNS resolver: OPNsense's Unbound for now (upgraded in Phase 3)

## Implementation sequence

1. **Prep** — download OPNsense ISO, verify SHA256, write to USB with `dd` / Rufus.
2. **Dry run in Packet Tracer** (study track Lab 08 preview) — model the WAN → firewall → LAN topology; run NAT overload; confirm you understand the packet path before touching hardware.
3. **Install on the appliance** — plug in keyboard + monitor (or serial console); install to internal SSD; assign interfaces (label physical ports: WAN, LAN, OPT1/mgmt, OPT2/unused).
4. **Initial config via LAN**:
   - Set root password (store in password manager only)
   - Change LAN IP to `172.27.100.1/24`
   - DHCP server on LAN: pool `172.27.100.100 – .199`
   - Enable HTTPS mgmt on LAN
5. **Firmware update** — apply pending updates before doing anything else.
6. **DNS**: OPNsense Unbound resolver, forwarders to `1.1.1.1` and `9.9.9.9` initially. Phase 3 replaces this with internal DNS.
7. **NTP**: OPNsense as NTP server for the LAN.
8. **Backup config** — export XML config, save to password manager or encrypted store. **This is your recovery-time saver.**
9. **Cutover** (maintenance window):
   - Power off ISP router (or put in bridge mode)
   - Plug ISP ONT → OPNsense WAN
   - Plug OPNsense LAN → household switch (temporarily the ISP router acting as dumb switch, or a cheap unmanaged switch)
   - Verify family devices get DHCP off OPNsense
   - Verify Internet works on 2–3 devices before leaving the room
10. **Basic firewall rules**:
    - LAN → WAN: allow all (default)
    - LAN → LAN: allow (single VLAN currently)
    - WAN → LAN: block all except OPNsense-managed states
    - No port-forwards yet
11. **Enable logging** — everything to local disk; syslog forwarding waits for Phase 5.
12. **Document** — `../runbooks/opnsense-baseline.md` with actual settings used, `../current-state.md` update, CHANGELOG entry.

## CCNA topics reinforced

- Static vs. dynamic routing (default route to WAN)
- NAT / PAT (source NAT for LAN → WAN)
- DHCP server operation
- Firewall zones and stateful inspection
- ACL fundamentals (allow/deny, direction, order-of-evaluation)
- DNS resolver architecture

## Study track lab (Packet Tracer)

**Lab 08 — NAT / PAT**. Build: [LAN switch — R1 (outside/inside NAT) — Cloud]. Configure PAT with `ip nat inside source list ... interface Gi0/0/1 overload`. Verify with `show ip nat translations`. This mirrors what OPNsense does automatically.

## Physical Oikos lab exercise

Once the cutover completes:

1. From your workstation, SSH to OPNsense (`ssh root@172.27.100.1`).
2. On OPNsense console: `configctl interface list`, `pfctl -sr | head -30`.
3. From a family laptop: `ping 172.27.100.1`, `ping 1.1.1.1`, `nslookup google.com`, `traceroute 1.1.1.1`. Explain each hop.
4. Trigger a firewall log: try to SSH into OPNsense from WAN side (should be blocked); observe the drop in Firewall → Log.

## Break/fix drill

Pick a low-family-usage moment:

1. Set a "wrong" LAN IP on OPNsense — save-and-reboot. LAN is now unreachable from your workstation.
2. Recover via **console cable + serial terminal** (`screen /dev/tty.usbserial* 115200` on macOS). Login as root, run `option 2` in the console menu, restore LAN IP.
3. Verify SSH returns.

Notes: this is the drill that saves you at 11 PM some future night. Do it once now while the household hasn't noticed.

## Verification commands

OPNsense (via SSH):

```bash
configctl interface list
configctl firewall list
pfctl -sr
pfctl -ss                     # active state table
dig @127.0.0.1 google.com
tail -f /var/log/filter.log   # live firewall drops
```

Family LAN client:

```bash
ip addr                # DHCP address in 172.27.100.100-199 range
ip route               # default via 172.27.100.1
nslookup google.com    # answered by 172.27.100.1
traceroute 1.1.1.1     # first hop = OPNsense
```

## Security posture at this phase

- **Mgmt UI**: HTTPS only, strong root password, LAN-side only, no WAN exposure ever
- **SSH**: keys-only, disabled from WAN
- **UPnP**: disabled
- **DNS rebind protection**: on
- **Anti-lockout rule**: keep enabled until Phase 2 (once mgmt VLAN is real you can tighten)
- **Backup**: config XML saved outside OPNsense (password manager or encrypted git-crypt)

## Monitoring coverage

- OPNsense's built-in dashboard + Reporting → Traffic Graph (basic)
- Firewall log local retention: 30 days min
- No Prometheus yet (Phase 5)

## Kephaleos docs to create / update

- **Create**: `work/projects/oikos/runbooks/opnsense-baseline.md` (settings actually used)
- **Create**: `work/projects/oikos/runbooks/opnsense-config-backup-restore.md`
- **Create**: `work/projects/oikos/troubleshooting/opnsense-lockout-recovery.md` (from the break/fix drill)
- **Update**: `current-state.md` — OPNsense DEPLOYED, VLAN 10 + 100 status changes
- **Update**: `inventory/nodes.md` — add firewall appliance
- **Update**: `architecture/address-plan.md` — VLAN 100 confirmed active
- **Append**: `CHANGELOG.md`
- **New ADR** if you deviate from Protectli/Qotom/CWWK class: `adr/000N-firewall-hardware.md`

## Diagram updates

- Master architecture diagram: ISP → OPNsense → LAN switch → clients (replaces old ISP-router-only path)
- Add "current-state" mini-diagram to `current-state.md`

## Completion checklist

- [ ] Firewall appliance installed and racked/placed
- [ ] OPNsense installed, firmware current
- [ ] Root password rotated and stored in password manager
- [ ] Config backup exported and stored securely
- [ ] Console cable tested (you know it works)
- [ ] Family devices get DHCP from OPNsense and reach Internet
- [ ] Firewall rules baseline documented
- [ ] Break/fix lockout-recovery drill completed
- [ ] Runbooks written
- [ ] `current-state.md`, `inventory/nodes.md`, `CHANGELOG.md` updated
- [ ] Master diagram updated
