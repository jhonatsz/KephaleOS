---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, architecture, conventions, naming]
confidence: high
---

# Naming Conventions

## Hostnames

Pattern: `<role>-<hw-tag>-<index>.oikos.home.arpa`

- **role** — lowercase, short (`pve`, `dc`, `nas`, `k8s-cp`, `k8s-worker`, `ai`)
- **hw-tag** — physical machine identifier (`t14`, `p520`); omit once the role is stable and hardware-independent
- **index** — zero-padded to 2 digits (`01`, `02`)

Examples:

- `pve-t14-01` — Proxmox on ThinkPad T14 (bootstrap)
- `pve-01` — future primary Proxmox, hardware-agnostic
- `dc01`, `dc02` — domain controllers (no hw-tag; role stable)
- `k8s-cp-01`, `k8s-worker-01`
- `nas01`, `ai01`

## VLANs

Short lowercase names matching the [[address-plan]]: `mgmt`, `servers`, `k8s`, `storage`, `ai`, `iot`, `camera`, `guest`, `home`. Interface labels on switches and firewall must match.

## Ansible

- **Inventory groups**: kebab-case matching role (`proxmox_nodes`, `k8s_control_plane`, `k8s_workers`, `nas`)
- **Variables**: `snake_case`
- **Roles**: singular, kebab-case (`proxmox-base`, `opnsense-baseline`, `pihole`)

## Terraform / OpenTofu

- **Modules**: `terraform/modules/<name>/`, one purpose per module
- **Resource names**: descriptive, no `main`/`default` catch-alls
- Split any `main.tf` beyond ~200 lines by concern

## ADRs

- **Staging while a phase is in flight**: `work/projects/oikos/adr/`
- **Durable, promoted**: `wiki/decisions/oikos-NNNN-<slug>.md` (4-digit sequence, always prefixed `oikos-` to namespace against other projects)
- **Never freehand** — invoke `/kephaleos-decision`

## Diagrams

- Prefer Mermaid inline in the Markdown page for small/medium diagrams
- Standalone Mermaid pages under `work/projects/oikos/diagrams/`
- Polished visual exports (draw.io / excalidraw / Figma) only for the master-architecture and multi-floor physical diagrams
- Every diagram declares which phase state it represents (current / next / final)
