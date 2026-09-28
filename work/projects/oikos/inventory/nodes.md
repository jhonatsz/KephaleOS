---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, inventory, hardware]
confidence: high
time-sensitive: true
---

# Node Inventory

Every planned or deployed compute node. Details per node live in `../nodes/<hostname>.md`.

| Node | Status | Hardware | Role | Mgmt IP | Phase | Detail |
|---|---|---|---|---|---|---|
| `pve-t14-01` | Hardware DEPLOYED, Proxmox IN PROGRESS | ThinkPad T14 Gen 2 · 40 GB · 4C/8T · 512 GB SSD | Proxmox (bootstrap) | 172.27.10.11 | 0 | [[../nodes/pve-t14-01]] |
| `pve-01` | PLANNED | TBD (candidate: Lenovo ThinkStation P520) | Primary Proxmox | 172.27.10.10 | 8 | — |
| `dc01` | PLANNED | VM (on `pve-t14-01`) | AD DC | 172.27.20.10 | 8 | — |
| `dc02` | PLANNED | VM (on `pve-01` — different physical host from dc01) | AD DC | 172.27.20.11 | 8 | — |
| `nas01` | PLANNED | TBD (ZFS-capable) | NAS | 172.27.40.x | 7 | — |
| `ai01` | PROPOSED | TBD (24 GB VRAM class GPU, 64+ GB RAM) | AI / LLM host | 172.27.50.x | 10 | — |

## Hardware on hand

- Lenovo ThinkPad T14 Gen 2 (40 GB RAM · 4C/8T · 512 GB SSD)

## Under evaluation (BUY WAIT)

- Lenovo ThinkStation P520 or equivalent as future primary Proxmox / GPU-capable node

## Not yet purchased (BUY per phase)

- Firewall appliance (Phase 1)
- Managed L2/L3 switch with 802.1Q + PoE (Phase 2)
- Wi-Fi APs, floor switches, structured cabling (Phase 4)
- UPS (Phase 5 or earlier if power draw warrants)
- NAS + drives + off-site backup target (Phase 7)
- Second Proxmox host (Phase 8)
- GPU / AI node (Phase 10)
- Rack, patch panel (Phase 4)
