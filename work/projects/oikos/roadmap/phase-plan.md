---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, planning, ccna]
confidence: high
---

# Oikos — Phase Plan

Redesigned phase sequence, ordered by **dependency** and **CCNA learning progression** — not by the original prompt's list order.

## Structure

**Numbered phases = real Oikos infrastructure changes.** They advance one at a time, each has a completion checklist, and no phase begins until the prior phase's checklist is done.

**Study track = parallel CCNA lab workstream.** Runs continuously in the background from day 1 in Packet Tracer, joined later by EVE-NG on Proxmox. Not numbered. Not gated. See [[../study/README|study/]].

## Guiding principles

1. **Networking fundamentals before abstractions.** Ethernet → IP → VLANs → routing → services → identity → orchestration → AI. Do not let virtualization or Kubernetes hide fundamentals I haven't yet learned.
2. **Household reliability is non-negotiable.** The production path (OPNsense, core switch, DNS, Wi-Fi) never carries CCNA experiments.
3. **Cheap wins first.** Reuse the T14. Buy hardware just-in-time per phase.
4. **Docs and monitoring track the build**, not trail it.
5. **Every phase is also a CCNA lesson** — CCNA topics, a Packet Tracer/EVE-NG lab in the study track, a real-lab exercise, and a break/fix drill.

## Phase map (executive summary)

| # | Phase | Prereqs | Household risk | Buy trigger |
|---|---|---|---|---|
| 0 | Foundation — docs, address plan, Proxmox on T14 | — | none | none |
| 1 | **Production firewall — OPNsense + first VLAN split** | 0 | HIGH (gateway swap) | firewall appliance |
| 2 | Managed switching & full VLAN rollout | 1 | medium | L3 managed switch |
| 3 | Network services — DNS, DHCP, NTP, filtering | 2 | medium | none |
| 4 | Wi-Fi + multi-floor distribution | 2, 3 | medium | APs, floor switches, structured cabling |
| 5 | Observability foundation — Prometheus, Grafana, syslog | 3 | low | UPS if not sooner |
| 6 | Advanced CCNA lab — EVE-NG/GNS3 on Proxmox | 3, study track | none | more RAM if T14 tight |
| 7 | Storage — NAS + backups | 3, 5 | low | NAS + drives + off-site target |
| 8 | Second Proxmox node + identity (DC01/DC02) | 7 | low | 2nd Proxmox host |
| 9 | Kubernetes — single cluster, multi-node | 7, 8 | low | none (VMs on Proxmox) |
| 10 | AI/GPU — local LLMs, LiteLLM gateway | 7, 9 | low | GPU node |
| 11 | Automation maturity — GitOps, IaC coverage, secrets mgmt | ongoing | low | none |

## Why this order (rationale)

- **Phase 1 (OPNsense) must precede everything else on the real network.** No VLAN, no DNS filtering, no monitoring is safe without a proper firewall boundary. This is the single scariest change for the household — the "swap the gateway" event. It comes first among the real infra phases *because* everything else assumes segmentation exists.
- **Phase 2 (switching + VLANs) must precede Phase 4 (Wi-Fi).** You can't design multi-floor distribution without knowing how trunks and access ports work.
- **Phase 5 (observability) lands before storage/identity/k8s** so every subsequent phase gains visibility from day 1. Adding monitoring later means half your infra is a black box.
- **Phase 6 (EVE-NG) waits** until foundational compute + services exist — nested virtualization + 4–8 GB per virtual router demands real capacity. Packet Tracer covers everything until then via the study track.
- **Phase 8 (identity) waits for two hosts.** A DC pair on one physical host buys no availability. AD DNS integration is also cleaner when internal DNS (Phase 3) is stable and you know exactly what it owns.
- **Phase 9 (k8s) waits for storage.** Persistent volumes without a NAS are frustrating and unrealistic.
- **Phase 10 (AI) is last** — depends on storage (models), network (LLM gateway routing), and ideally k8s for serving. Also the single biggest capex line.
- **Phase 11 (automation)** is a horizontal thread. Ansible starts in Phase 0. GitOps arrives with k8s. IaC coverage grows every phase. Do not automate what you haven't first configured manually.

## What changed vs. the original prompt

- **AI (originally item 9) moved to Phase 10**, not Phase 3–4. It has almost no useful signal until the underlying platform exists.
- **Identity (DC01/DC02) moved to Phase 8**, not near the front. Two DCs on a single Proxmox node give false redundancy; wait for a second physical host.
- **Observability moved earlier** (Phase 5) than the prompt implied. Cheap to add, expensive to add late.
- **Kubernetes moved to Phase 9**, gated on storage.
- **Packet Tracer is a parallel study track**, not a numbered phase. Starts alongside Phase 0.
- **EVE-NG is its own phase (6)** rather than "when convenient" — it's a real capacity commitment.

## Per-phase pages (all drafted)

- [[phase-00-foundation]] *(in progress — you are here)*
- [[phase-01-opnsense]]
- [[phase-02-switching-vlans]]
- [[phase-03-network-services]]
- [[phase-04-wifi-multifloor]]
- [[phase-05-observability]]
- [[phase-06-eve-ng]]
- [[phase-07-storage]]
- [[phase-08-second-node-identity]]
- [[phase-09-kubernetes]]
- [[phase-10-ai]]
- [[phase-11-automation-maturity]]

Companion:

- [[hardware-roadmap]] — consolidated hardware/materials list with cost bands and buy timing

Study track:

- [[../study/README|study/README]]
- [[../study/packet-tracer|study/packet-tracer]] *(labs 01–10, starts immediately)*

Each phase page follows this template:

- Objective / why now / prerequisites
- Hardware (required vs. optional, buy timing)
- Software
- Network changes
- Implementation sequence
- CCNA topics reinforced
- Packet Tracer / EVE-NG lab exercise (from the study track)
- Physical Oikos lab exercise
- Break/fix drill
- Verification commands
- Security posture at this phase
- Monitoring coverage at this phase
- Kephaleos docs to create/update
- Diagrams that must change
- Completion checklist

## Right-now action

Phase 0. Concretely:

1. ✅ Docs scaffold
2. ✅ Proxmox installed on the T14 (in progress by operator)
3. Apply hostname fix per [[../../../../wiki/decisions/oikos-0001-dns-namespace|ADR oikos-0001]]
4. Post-install hygiene (repos, updates, SSH keys, root password rotation)
5. First Debian cloud-init VM template
6. Save initial config snapshot to `~/workspace/personal/homelab/`
7. Break/fix drill
8. In parallel: kick off the study track — install Packet Tracer, complete Lab 01

**Do not touch the household network yet.** That is Phase 1.
