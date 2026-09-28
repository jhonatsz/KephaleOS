---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
aliases: [Oikos, oikos-homelab, oikos.home.arpa]
tags: [project, homelab, oikos, networking, proxmox, ccna, self-hosted]
repo: ~/workspace/personal/homelab/
confidence: high
---

# Project: Oikos

**Oikos** is my long-term homelab — a small production-style infrastructure environment that doubles as a hands-on lab for CCNA study, Proxmox, security, storage, Kubernetes, DevOps, and local AI. Greek οἶκος = "household," fitting for infra that must serve the family reliably while I learn on it.

**Repo (code/config)**: `~/workspace/personal/homelab/`
**Operational docs (in-flight)**: [[../../work/projects/oikos/README|work/projects/oikos/]]

## What Oikos is

Single-operator homelab designed around the five-hat model: production household infrastructure (family internet, Wi-Fi, DNS, firewall), a CCNA-first networking lab, virtualization and platform engineering (Proxmox → Kubernetes), storage (NAS + backups), and local AI (LLM gateway on GPU). See the [[../../work/projects/oikos/roadmap/phase-plan|phase plan]] for the sequenced build.

## Hard invariants

- **Safe-lab principle** — household production never carries CCNA experiments. Packet Tracer / EVE-NG / GNS3 are the destructive playground.
- **DEPLOYED / CONFIGURED / PLANNED / PROPOSED / DEPRECATED / RETIRED** — every artifact declares status; future state is never described as present.
- **Manual before automation** — learn → document → repeat → automate. No Ansible for a service before it has been hand-configured and troubleshot at least once.
- **Docs are part of the change** — current-state, inventory, IPAM, diagrams, ADR, CHANGELOG all update alongside every infra change.

## Namespace & addressing

- Internal DNS: `oikos.home.arpa` (RFC 8375 reserved — no public-DNS collision)
- Address space: `172.27.0.0/16`
- Hostname pattern: `<role>-<hw-tag>-<index>.oikos.home.arpa` (e.g. `pve-t14-01`)

Full VLAN/subnet plan: [[../../work/projects/oikos/architecture/address-plan]].

## Current state (short)

Only the ThinkPad T14 Gen 2 hardware exists. **Phase 0** in progress: docs scaffold + Proxmox install on the T14. Live snapshot: [[../../work/projects/oikos/current-state]].

## Related canonical pages (grown as needed)

- `wiki/technologies/proxmox.md`
- `wiki/technologies/opnsense.md`
- `wiki/technologies/vlans.md`
- `wiki/runbooks/proxmox-install.md`
- `wiki/decisions/oikos-*` — Oikos ADRs
