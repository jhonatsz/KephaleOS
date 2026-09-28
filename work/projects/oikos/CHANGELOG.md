# Oikos CHANGELOG

Dated log of infrastructure changes. Newest first. Every non-trivial change lands here.

## 2026-09-18 (night)

- **Drafted all 11 remaining phase pages** (Phase 1–11): OPNsense, switching/VLANs, network services, Wi-Fi/multi-floor, observability, EVE-NG, storage, second node + identity, Kubernetes, AI/GPU, automation maturity. Each follows the standard template (objective / why now / prereqs / hardware / software / network changes / implementation sequence / CCNA topics / study lab / physical lab / break-fix drill / verification / security / monitoring / docs to update / diagrams / completion checklist).
- **Consolidated hardware roadmap**: `roadmap/hardware-roadmap.md` — every item Oikos needs, grouped by phase, with required/optional, buy timing, cost bands, and cost-avoidance principles.

## 2026-09-18 (evening)

- **Restructured phase plan**: Packet Tracer moved out of the numbered phases into a parallel [[study/README|study track]]. Phases renumbered so **Phase 1 = OPNsense/firewall** (was Phase 2), matching operator mental model. All downstream phases shifted by −1. Total numbered phases now 0–11 (was 0–12). Rationale: labs are a continuous parallel workstream, not a discrete infra phase.
- **ADR oikos-0001 accepted**: internal DNS zone is `oikos.home.arpa` (RFC 8375-compliant). Promoted to [[../../../wiki/decisions/oikos-0001-dns-namespace|wiki/decisions/oikos-0001-dns-namespace]]; local stub at `adr/0001-dns-namespace.md` marks superseded. Operator applying `hostnamectl` correction on the in-flight T14 install.
- **Phase 0 in flight — Proxmox install started on the T14.**
- **Wrote Phase 0** (`roadmap/phase-00-foundation.md`) and the Packet Tracer study track (`study/README.md`, `study/packet-tracer.md`) full pages: objective, prereqs, implementation sequence, CCNA topics, break/fix drill, verification, security, monitoring, docs, completion checklist.
- Updated `current-state.md` and `nodes/pve-t14-01.md` to reflect install-in-progress + namespace flag.

## 2026-09-18 (morning)

- **Bootstrapped Oikos documentation scaffold** in Kephaleos (`work/projects/oikos/` + `wiki/projects/oikos.md`) and in the homelab repo (`~/workspace/personal/homelab/CLAUDE.md`, `README.md`, `.gitignore`). Kephaleos placement follows the vault charter's `work/` vs. `wiki/` split — not the top-level `projects/oikos/` implied by the original prompt.
- **Redesigned the phase plan** into 12 numbered phases (0–11) plus a parallel study track, ordered by dependency and CCNA learning progression. See [[roadmap/phase-plan]] for rationale and per-phase pages.
- **Committed address plan**: `172.27.0.0/16` with 9 VLANs; internal DNS namespace `oikos.home.arpa` (pending ADR 0001).
- **Inventory baseline**: Lenovo ThinkPad T14 Gen 2 is the only hardware on hand. Everything else PLANNED or PROPOSED.
