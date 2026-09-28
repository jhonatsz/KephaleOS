---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
aliases: [oikos-work, oikos-ops]
tags: [oikos, homelab, ops, in-flight]
---

# Oikos — Operational Hub

In-flight operational context for the Oikos homelab. Canonical single-page overview: [[../../../wiki/projects/oikos|wiki/projects/oikos]]. Code and config: `~/workspace/personal/homelab/`.

## Start here

1. [[current-state]] — what actually exists right now
2. [[roadmap/phase-plan|Phase plan]] — sequenced build, redesigned by dependency
3. [[architecture/address-plan|Address & VLAN plan]]
4. [[architecture/naming-conventions|Naming conventions]]
5. [[inventory/nodes|Hardware inventory]]
6. [[nodes/pve-t14-01|pve-t14-01 (ThinkPad Proxmox)]]

## Directory map

- `architecture/` — address plan, naming, project-scoped design notes
- `diagrams/` — Mermaid sources (master, VLAN, physical, per-phase)
- `inventory/` — nodes, IPAM tracker, cabling
- `nodes/` — per-node build details
- `network/` — per-VLAN, per-service network notes
- `runbooks/` — project-scoped runbooks (promote cross-cutting ones to `wiki/runbooks/`)
- `troubleshooting/` — symptom → diagnosis
- `adr/` — project-scoped ADR staging (promote durable ones to `wiki/decisions/oikos-NNNN-*`)
- `security/` — threat model, security invariants, incident log
- `backup/` — backup strategy, restore runbooks, verification cadence
- `monitoring/` — dashboards, alert catalog, SLIs
- `roadmap/` — phase plan and per-phase pages
- `CHANGELOG.md` — dated infrastructure changes

## Obsidian workspace setup

Use Obsidian's core **Workspaces** plugin (not separate vaults) to switch context. Recommended saved workspaces:

- **Kephaleos — Personal** (default): full vault, dashboard focus
- **Kephaleos — Oikos**: file explorer scoped to `work/projects/oikos/` + `wiki/projects/oikos.md`, Mermaid preview pane, graph filtered to `path:oikos`

Load via `Cmd/Ctrl+P → Workspaces: Load layout`. One vault, one graph, backlinks intact — psychological separation without breaking `/kephaleos-recall` or the charter.

To create: open the file tree the way you want it, then `Cmd/Ctrl+P → Workspaces: Save layout`. Assign a hotkey in Settings → Hotkeys → Workspaces.

## Living-doc rule

Every infrastructure change flows through:

```
change → current-state → inventory / IPAM → architecture → diagrams → ADR → CHANGELOG
```

Docs are part of the work, not after it.

## Promotion to `wiki/`

When something in this tree becomes durable and cross-cutting:

- Runbook that applies beyond Oikos → move to `wiki/runbooks/`
- Decision with generalizable reasoning → promote to `wiki/decisions/oikos-NNNN-*`
- Technology page (e.g., how Proxmox actually works) → move to `wiki/technologies/`

Leave a stub link behind so this tree stays navigable.
