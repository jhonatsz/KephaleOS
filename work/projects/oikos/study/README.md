---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, study, ccna, lab, packet-tracer, eve-ng]
confidence: high
---

# Oikos — Study Track

Parallel workstream for CCNA / networking study. **Not a phase.** Runs continuously from day 1 alongside the numbered infrastructure phases in [[../roadmap/phase-plan]].

## Why it's separate

CCNA labs happen in a sandbox (Packet Tracer, then EVE-NG) that never touches household or Oikos production. That makes them:

- **Never gated** by infra readiness
- **Never a household risk**
- **Always available** for prototyping the next phase before you build it for real

Treating them as their own numbered phase over-counted work and confused the sequence. This track lives here instead.

## Track components

| Component | Kicks off | Notes |
|---|---|---|
| [[packet-tracer\|Packet Tracer]] | Alongside Phase 0 | Free, runs on workstation, covers 90% of CCNA topology drills |
| [[eve-ng\|EVE-NG / GNS3]] | Once Phase 6 provisions it on Proxmox | Real Cisco IOS images; needed for OSPF/BGP realism and advanced troubleshooting |
| CCNA reading | Continuous | Cisco NetAcad, Odom Official Cert Guide, Boson practice exams |
| Lab notes | Every session | Every `.pkt` gets a `notes.md`. If it's not documented, it didn't happen |

## Cadence

- **1 lab or reading session per weekday minimum.** Small and frequent > big and rare.
- **Every Oikos phase** has a corresponding lab in the study track — prototype in Packet Tracer *before* touching real hardware.
- **Every 4–6 weeks**: sit a Boson-style practice exam to calibrate.

## Rule

Whenever a phase is about to introduce a networking concept (VLANs, trunks, OSPF, ACLs, NAT), the corresponding lab must be *completed and understood in the study track first*. Not just watched. Built, verified, broken, fixed, documented.

## Related

- [[packet-tracer]] — the labs index and Lab 01
- [[../roadmap/phase-plan]] — how the study track maps to each phase
- `~/workspace/personal/homelab/labs/` — where `.pkt` files live in the repo
