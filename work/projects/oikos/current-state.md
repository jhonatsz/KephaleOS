---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, current-state, inventory]
confidence: high
time-sensitive: true
---

# Oikos — Current State

Live snapshot of what exists. **Update alongside every change.** If the code/config surface and this page disagree, this page is wrong until you fix it.

## Status legend

| Status | Meaning |
|---|---|
| DEPLOYED | Physically installed and running |
| CONFIGURED | Installed but not yet in production use |
| PLANNED | Design decided, not yet executed |
| PROPOSED | Under evaluation, no commitment |
| DEPRECATED | Being retired, still running |
| RETIRED | Removed |

## Hardware

| Item | Model | Specs | Status | Phase | Notes |
|---|---|---|---|---|---|
| Compute node 1 | Lenovo ThinkPad T14 Gen 2 | 40 GB RAM, 4C/8T, 512 GB SSD | Proxmox install IN PROGRESS | 0 | Being installed 2026-09-18 as `pve-t14-01.oikos.home.arpa` per [[../../../wiki/decisions/oikos-0001-dns-namespace\|ADR oikos-0001]] |
| Firewall appliance | — | — | PLANNED | 1 | Candidate: Protectli/Qotom mini-PC dual-NIC |
| Core managed switch | — | — | PLANNED | 2 | L2/L3 with 802.1Q + PoE |
| Access points (~5) | — | — | PLANNED | 4 | One per floor target |
| Floor switches | — | — | PLANNED | 4 | If PoE APs need injectors |
| NAS + drives | — | — | PLANNED | 7 | ZFS-capable |
| Second Proxmox node | — | — | PLANNED | 8 | Candidate: Lenovo ThinkStation P520 |
| GPU / AI node | — | — | PROPOSED | 10 | 24 GB VRAM class |
| UPS | — | — | PROPOSED | 5 or earlier | Sizing depends on rack draw |
| Rack / patch panel / cabling | — | — | PLANNED | 4 | Structured cabling per floor |

## Software / services

Nothing installed yet.

## Network segmentation

| VLAN | Name | CIDR | Status |
|---|---|---|---|
| 10 | mgmt | 172.27.10.0/24 | PLANNED |
| 20 | servers | 172.27.20.0/24 | PLANNED |
| 30 | k8s | 172.27.30.0/24 | PLANNED |
| 40 | storage | 172.27.40.0/24 | PLANNED |
| 50 | ai | 172.27.50.0/24 | PLANNED |
| 60 | iot | 172.27.60.0/24 | PLANNED |
| 70 | camera | 172.27.70.0/24 | PLANNED |
| 80 | guest | 172.27.80.0/24 | PLANNED |
| 100 | home | 172.27.100.0/24 | PLANNED |

## Reserved addresses

| Address | Host | Status |
|---|---|---|
| 172.27.10.1 | Firewall / mgmt gateway | PLANNED |
| 172.27.10.10 | `pve-01` (future primary Proxmox) | PLANNED |
| 172.27.10.11 | `pve-t14-01` | PLANNED |
| 172.27.20.10 | `dc01` | PLANNED |
| 172.27.20.11 | `dc02` | PLANNED |

## Household network today (baseline before Oikos)

ISP-provided router/firewall in default configuration; no VLANs; single flat Wi-Fi SSID; no internal DNS. This is what Phase 2 replaces without downtime for the family.

## Right-now next step

**Phase 0**: install Proxmox on the T14 with parameters from [[nodes/pve-t14-01]]. Kick off the **[[study/packet-tracer|Packet Tracer study track]]** in parallel. Do NOT touch the household network — that is Phase 1.
