---
type: dashboard
updated: 2026-09-13
---

# Knowledge Map

Map of what Kephaleos knows — not a file index. Updated only when a new
important domain, cluster, synthesis, or gap emerges.

---

## Core domains

Expected growth areas — plain text, not wikilinks, until seeded with real
canonical pages (avoiding broken-link noise for lint):

`AI` · `Cloud Infrastructure` · `DevOps` · `Cybersecurity` ·
`Software Engineering` · `Business` · `Personal`

---

## Active knowledge clusters

### Kubernetes / EKS operations

- [[ccsi-msd-prd EKS cluster]] — shared prod cluster; systematic CPU
  over-reservation; no autoscaler; no `Pending`-pod alerting
- [[HPA memory-requests pin at max]] — memory-HPA pitfall; steady-state
  workloads pin at max when `requests.memory` sits near real usage
- [[Incident: GitLab Runner CPU reservation]] (`work/incidents/`) — the
  first documented instance of the HPA-driven CPU reservation exhaustion
- **Latent synthesis:** if 2+ more incidents show CPU-request over-provisioning
  or memory-HPA misconfiguration, promote to
  `[[Kubernetes Deployment Readiness Checklist]]`.

### macOS networking / VPN

- [[Solution: VPN fails when another VPN/ZTNA agent installed]] — Zscaler
  Z-Tunnel steals per-host routes for the outer VPN's gateway; diagnosis
  via `route -n get <ip>`; CGNAT range (100.64.0.0/10) as fingerprint

### Incident communication

- [[Outage channel announcement runbook]] — CyberSoft `! IMP - Outages`
  channel format for live incident + inline Sev-3 postmortem

### Kephaleos itself

- [[CLAUDE.md]] — Phase 1.1 knowledge-maintainer charter
- [[Learning: Kephaleos must be accessible from any working directory]]

---

## Important syntheses

*(none yet — one is latent: Kubernetes deployment readiness.)*

---

## Current knowledge gaps

- **Proton VPN + FortiClient behavior.** Proton's WireGuard extension is
  present but inactive on the user's machine. If enabled, a similar
  routing conflict is possible. No capture yet.
- **Right-sizing follow-up on `ccsi-msd-prd`.** Follow-ups from the
  GitLab runner incident (raise `sbiqai` `requests.memory`, lower
  `requests.cpu`, extend review to `tasktile` / `cybersoft-dtr` /
  `adminsynonyms`, add `Pending`-pod alert) — outcomes not yet captured.
- **Zscaler bypass ticket outcome.** Whether IT added the FortiGate IP
  to the ZPA bypass list — durable fix is not yet confirmed.
- **Decisions folder is empty.** No ADR-style decisions captured yet;
  the highest-value knowledge type (charter §18) has zero entries.

---

## How to grow this map

Update `knowledge-map.md` only when:

- an important knowledge domain emerges
- a significant cluster forms (3+ related canonical pages)
- an important synthesis is written
- a major knowledge gap becomes apparent
- the overall knowledge structure materially changes

Do NOT mechanically add every new wiki page.
