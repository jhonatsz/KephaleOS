---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, study, ccna, packet-tracer, lab]
confidence: high
---

# Study Track — Packet Tracer

Cisco Packet Tracer is the primary sandbox for the study track. Runs on any workstation, free (with Cisco NetAcad account), covers 90% of CCNA topology drills. Kicks off **alongside Phase 0** and runs continuously.

Lab `.pkt` files live in the homelab repo: `~/workspace/personal/homelab/labs/packet-tracer/NN-slug/`. Every lab has a `notes.md` next to it.

## Setup (one-time)

1. Register for a free Cisco NetAcad account (netacad.com).
2. Enroll in **Introduction to Packet Tracer** — enrolls you and unlocks the download.
3. Install Packet Tracer (latest — 8.2+ as of 2026-09).
4. Complete the built-in "Getting Started with Packet Tracer" tutorial.
5. Create the labs directory in the homelab repo (already scaffolded at `~/workspace/personal/homelab/labs/packet-tracer/`).

## Habits (every lab)

- Enable secret on every device — `enable secret <password>` — even in a sandbox
- `line console 0` + `line vty 0 4` with `password` and `login`
- `service password-encryption`
- `copy running-config startup-config` before closing the file
- Write `notes.md` alongside `.pkt`: what I built, commands used, what I broke, why

## Verification commands (memorize)

```
show ip interface brief         # you'll type this 10,000 times before CCNA passes
show running-config
show startup-config
show version
show mac address-table          # switch
show arp                        # router
show interfaces status          # switch
show vlan brief                 # switch
show interfaces trunk           # switch
show cdp neighbors
show ip route
show spanning-tree
show etherchannel summary
show ip ospf neighbor
ping <ip>
traceroute <ip>
```

## Labs index

| # | Lab | Phase alignment | Status | Focus |
|---|---|---|---|---|
| 01 | CLI basics + single subnet | Phase 0 | not started | IOS CLI, single subnet, PC↔PC via switch |
| 02 | Switch VLANs | Phase 2 | not started | Access VLANs, MAC-table isolation |
| 03 | 802.1Q trunks | Phase 2 | not started | Trunks, native VLAN, allowed VLAN list |
| 04 | Router-on-a-stick | Phase 2 | not started | Inter-VLAN routing via subinterfaces |
| 05 | Static + default routes | Phase 3 | not started | Static routes, `ip route 0.0.0.0` |
| 06 | DHCP relay | Phase 3 | not started | `ip helper-address` across routers |
| 07 | ACLs | Phase 3/5 | not started | Standard + extended ACLs, direction |
| 08 | NAT / PAT | Phase 1 refresher | not started | Static NAT, PAT overloading |
| 09 | OSPF single-area | Phase 6 (EVE-NG preferred) | not started | Neighbor states, `router-id`, `network` |
| 10 | STP/RSTP + EtherChannel | Phase 6 | not started | Root bridge election, port states, LACP |

---

## Lab 01 — CLI basics + single subnet

**Kicks off with Phase 0.** No prerequisites beyond Packet Tracer installed.

### Topology

```
[PC1] ─── [SW1: 2960] ─── [PC2]
              │
           [R1: 4331]
```

### Addressing

| Device | Interface | Address |
|---|---|---|
| PC1 | fa0 | 10.1.1.10/24 · GW 10.1.1.1 |
| PC2 | fa0 | 10.1.1.11/24 · GW 10.1.1.1 |
| R1 | Gi0/0/0 | 10.1.1.1/24 · `no shutdown` |
| SW1 | (default) | all ports access VLAN 1 |

### Success criteria

- PC1 ↔ PC2 ping succeeds (same subnet, learned via ARP + switching)
- PC1/PC2 → R1 ping succeeds (default gateway reachable)
- `show mac address-table` on SW1 shows all three MACs
- `show arp` on R1 shows both PCs
- `copy run start` persists config across a device reload

### Save

`01-cli-basics.pkt` + `notes.md` under `~/workspace/personal/homelab/labs/packet-tracer/01-cli-basics/`.

### Break/fix drill

1. On R1: `int Gi0/0/0` → `shutdown`. Observe: PC-to-router ping fails; PC-to-PC still works. **Why?** Answer in notes.
2. `no shutdown`. Verify recovery.
3. On PC1: change IP to `10.1.2.10/24` (wrong subnet), keep the same gateway. Observe: ping to PC2 fails. **Why does it fail even though they're on the same switch?** Answer in notes.
4. Restore.
5. On SW1: `int fa0/1` → `shutdown`. Which `show` command tells you the port is down?
6. `no shutdown`. Verify.
7. On R1: change enable secret. Log out. Log back in — did running-config survive? Did startup-config? What if you didn't `write memory`?

### CCNA topics reinforced

- Cisco IOS command hierarchy (user EXEC → privileged EXEC → global config → interface)
- `?` and tab completion
- Interface addressing
- `show` command family
- running-config vs. startup-config
- Basic switch operation: MAC learning, flooding, forwarding
- ARP behavior

### Completion checklist

- [ ] Packet Tracer installed
- [ ] NetAcad "Getting Started" tutorial done
- [ ] Lab 01 built cleanly, all four success criteria met
- [ ] Break/fix drill run; every "why" answered in `notes.md`
- [ ] `01-cli-basics.pkt` + `notes.md` committed to homelab repo
- [ ] `troubleshooting/ios-cli-cheatsheet.md` seeded (in `work/projects/oikos/troubleshooting/`)

---

Further labs (02+) will be written out at the top of the file as we approach each phase. Structure: same as Lab 01 — topology, addressing, success criteria, break/fix, topics reinforced, checklist.
