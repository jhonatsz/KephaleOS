---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, architecture, network, ipam, vlan]
confidence: high
---

# Address & VLAN Plan

## Private address space

**Oikos**: `172.27.0.0/16` — deliberately outside common defaults (`192.168.0.0/16`, `10.0.0.0/8`) and typical corporate VPN spaces I'd overlap with.

## Segmentation

VLANs are the *logical* boundary. **Do not conflate them with physical floors** — a single floor can carry many VLANs on a trunk.

| VLAN | Name | CIDR | Purpose | Target egress policy |
|---|---|---|---|---|
| 10 | mgmt | 172.27.10.0/24 | Management plane (firewall, Proxmox mgmt, switch/AP mgmt) | Deny from all VLANs except operator hosts |
| 20 | servers | 172.27.20.0/24 | Core VMs (DCs, DNS, monitoring) | Restrictive; publish specific ports |
| 30 | k8s | 172.27.30.0/24 | Kubernetes node underlay | Pod/service overlays live inside |
| 40 | storage | 172.27.40.0/24 | NAS, backup targets | Servers/k8s/ai read; deny Internet |
| 50 | ai | 172.27.50.0/24 | GPU/LLM workloads | Storage read; controlled egress |
| 60 | iot | 172.27.60.0/24 | IoT | Internet allowed; deny inter-VLAN |
| 70 | camera | 172.27.70.0/24 | Security cameras | Deny Internet; NVR reachable from servers |
| 80 | guest | 172.27.80.0/24 | Guest Wi-Fi | Internet only; deny all RFC1918 |
| 100 | home | 172.27.100.0/24 | Family devices | Default household trust zone |

## DNS

- **Internal namespace**: `oikos.home.arpa` (RFC 8375 reserved)
- **Hostname pattern**: `<role>-<hw-tag>-<index>.oikos.home.arpa`

Examples: `pve-t14-01`, `pve-01`, `dc01`, `dc02`, `nas01`, `k8s-cp-01`, `k8s-worker-01`, `ai01`.

## Reserved addresses

| Address | Host | Status |
|---|---|---|
| 172.27.10.1 | Firewall / mgmt gateway | PLANNED |
| 172.27.10.10 | `pve-01` | PLANNED |
| 172.27.10.11 | `pve-t14-01` | PLANNED |
| 172.27.20.10 | `dc01` | PLANNED |
| 172.27.20.11 | `dc02` | PLANNED |

## Per-subnet conventions

- **Gateway**: `.1` in every subnet
- **Static assignments**: `.10-.99`
- **DHCP pool**: `.100-.199`
- **Reserved for future**: `.200-.254`

## Design rationale (why this shape)

- **Home (VLAN 100) separate from Mgmt (VLAN 10)** — a compromised family laptop must not reach the management plane. This is the single most important boundary in the whole design.
- **Cameras (70) separate from IoT (60)** — even though both are "untrusted appliances," they have different egress policy: cameras deny Internet by default, IoT often needs cloud.
- **No per-floor VLANs** — physical topology is orthogonal to logical segmentation. A camera on floor 3 belongs on VLAN 70, not "floor-3 VLAN."
- **VLAN 30 for k8s node IPs only** — pod/service CIDRs are overlay space (CNI-dependent), plan in Phase 9, keep out of the underlay plan.
- **VLAN 20 not merged with mgmt** — servers get their own segment so I can allow specific inbound service ports without opening the management plane.

## Open questions (revisit at their phase)

- **IPv6 from day 1 on mgmt?** — pushes IPv6 CCNA content earlier; benefits: dual-stack muscle memory. Cost: more moving parts during bootstrap. **Tentative**: defer to a dedicated IPv6 phase.
- **AI (50) → Storage (40) routing** — direct L3 (fast, less isolation) vs. via servers (slower, easier ACLs)? Revisit Phase 10 with real numbers.
- **Cameras on their own physical switch?** — pure L2 isolation vs. VLAN-only. Revisit Phase 2 when picking the switch.

## Related

- [[naming-conventions]]
- [[../roadmap/phase-plan]]
- [[../inventory/nodes]]
