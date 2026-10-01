---
type: dashboard
updated: 2026-10-01---

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

### CyberSoft (employer)

- [[CyberSoft]] — org page, 6-service catalog, offices, delivery platforms
- Projects: [[Yakap]] · [[CyberSoft SFTP]] · [[cybersoftbpo.com rebuild]] ·
  [[colpaliservice]] · [[aws-tf-network]] · [[k8s-gitlab-runner]]
- Shared prod: [[ccsi-msd-prd EKS cluster]]
- Runbooks: [[Outage channel announcement runbook]] ·
  [[Rotate GitLab Runner token]]
- **Open loop:** colpali secret-leak — credentials rotated 2026-09-30;
  local git-history rewrite prepared but not yet force-pushed

### Techstyle (employer)

- [[Techstyle]] — org page (Fabletics, Savage X); GitHub org `TechstyleOS`
- [[NorthStar]] — Apollo Router v2 IaC for the federated storefront
  gateway across us-west-2 / us-east-1 / eu-west-3
- [[duplo-prd-usw2 EKS cluster]] — US-West production topology snapshot

### Oikos (personal homelab)

- [[wiki/projects/oikos|Oikos]] — canonical project page
- ADR: [[wiki/decisions/oikos-0001-dns-namespace|oikos-0001 — internal DNS namespace `oikos.home.arpa`]]
- Full operational tree at [[work/projects/oikos/README|work/projects/oikos/]] —
  address plan (172.27.0.0/16, 9 VLANs), naming conventions, T14
  bootstrap node, 11 numbered phase pages (0–11), hardware roadmap,
  parallel study track (Packet Tracer + EVE-NG/GNS3)

### Kubernetes / EKS operations

- [[ccsi-msd-prd EKS cluster]] — shared prod (CyberSoft); systematic CPU
  over-reservation; no autoscaler; no `Pending`-pod alerting
- [[duplo-prd-usw2 EKS cluster]] — Techstyle prod; Apollo Router tenant +
  pricing services; time-sensitive topology snapshot
- [[HPA memory-requests pin at max]] — memory-HPA pitfall; steady-state
  workloads pin at max when `requests.memory` sits near real usage
- [[Requests-vs-usage divergence starves cluster capacity]] — the
  opposite calibration failure (requests too high → HPA suppressed)
- [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]]
  — CA can be healthy while managing zero node groups
- [[Incident: GitLab Runner CPU reservation]] (`work/incidents/`) — first
  documented instance of HPA-driven CPU reservation exhaustion
- **Synthesis:** [[Kubernetes Deployment Readiness Checklist]] — 6-item
  pre-deploy checklist compiled from the 3 learnings + the incident +
  the 2 cluster pages. Promoted from latent on 2026-10-01.

### AWS network infrastructure

- [[aws-tf-network]] — single-VPC (10.51.0.0/16, us-west-2), 4-env
  subnet tiers, VPN + peerings; 8 known issues documented
- [[AWS ALB]] / [[AWS NLB]] — [[aws-elbv2-alb-nlb|ELBv2 operational patterns]]
  (listener drift, NLB-in-front destroy order, TLS remediation history)
- [[.NET Framework TLS]] — Windows 2016 + `SchUseStrongCrypto` /
  `SystemDefaultTlsVersions`; TLS 1.3 upgrade blockers for legacy
  workloads (e.g. bullzip SQS consumer stall incident)

### macOS networking / VPN

- [[Zscaler ZPA route hijack]] — canonical solution page (formally
  [[Solution: VPN fails when another VPN/ZTNA agent installed]]).
  Backed by **four** lived incidents in ~20 days (FortiClient IPsec,
  partner SFTP TCP RST, AWS EC2 CI runner SSH silent timeout, MS Teams
  signaling after ISP switch). Covers **two distinct enforcement
  layers**: Mode A (FIB route hijack — fix with `route delete/add`)
  and Mode B (stale NetworkExtension claim after network context change
  — fix with `pkill -HUP ZscalerTunnel`). Third-occurrence rule
  triggered → escalation is now a ZPA policy scope review, not per-host
  bypass tickets.

### Incident communication

- [[Outage channel announcement runbook]] — CyberSoft `! IMP - Outages`
  channel format for live incident + inline Sev-3 postmortem
- [[Ops communication templates]] — portable maintenance / change /
  outage / postmortem templates with connected worked samples

### Kephaleos itself

- `CLAUDE.md` — Phase 1.1 knowledge-maintainer charter
- [[Learning: Kephaleos must be accessible from any working directory]]

---

## Important syntheses

- **Zscaler ZPA hijack pattern (4 incidents → one canonical solution, two
  modes).** Same machine, four unrelated destinations in ~20 days: corp
  FortiGate IPsec (UDP timeout), partner SFTP (fast TCP RST), employer
  EC2 CI runner SSH (silent timeout + ICMP), and MS Teams signaling
  after ISP switch (`EADDRNOTAVAIL` in <100 ms). Fingerprints:
  **Mode A (FIB hijack)** → `route -n get <ip>` shows CGNAT-range
  next-hop (100.64.0.0/10); fix with route override. **Mode B (stale
  NE)** → `route` is clean but socket bind returns `EADDRNOTAVAIL`
  immediately; fix with `pkill -HUP ZscalerTunnel`. GUI toggles for
  ZIA/ZPA do not clear Mode B — the stale claim lives in the system
  LaunchDaemon layer. Solution page carries the synthesis: escalate
  from per-host bypass tickets to a ZPA policy scope review; watch for
  stacked failure modes (ZPA hijack + SG allow-list mismatch) on
  employer-owned targets.
- **[[Kubernetes Deployment Readiness Checklist]]** (2026-10-01,
  promoted from latent). 6-item pre-deploy checklist that closes the
  compound failure mode of poorly-calibrated `requests` + broken
  elasticity mechanisms + missing pending-pod alerting. Compiled from
  3 learnings (HPA memory-requests, requests-vs-usage divergence, CA
  ASG discovery tags), 1 incident (GitLab runner CPU reservation),
  and 2 cluster pages (`ccsi-msd-prd`, `duplo-prd-usw2`).

---

## Current knowledge gaps

- **Proton VPN + FortiClient behavior.** Proton's WireGuard extension is
  present but inactive. If enabled, a similar routing conflict is
  possible. No capture yet.
- **Right-sizing follow-up on `ccsi-msd-prd`.** Follow-ups from the
  GitLab-runner incident (raise `sbiqai` `requests.memory`, lower
  `requests.cpu`, extend review to `tasktile` / `cybersoft-dtr` /
  `adminsynonyms`, add `Pending`-pod alert) — outcomes not yet captured.
- **ZPA policy scope review with IT — formal deferral recorded
  2026-10-02.** Promoted from a floating note on the solution page to
  [[wiki/decisions/2026-10-01-zpa-escalation-deferral|a recorded
  decision]] with full why, alternatives, consequences, and five explicit
  revisit triggers (recurrence rate, non-macOS-routable destination,
  user-facing miss, work-laptop migration slip past 2026-12-31, EMS
  scope broadening). This is no longer a gap — it's a documented decision
  with trigger conditions.
- **Colpali git-history rewrite.** Local force-rewrite prepared but not
  pushed. Credentials already rotated (safety net closed), so this is
  hygiene rather than risk — but the loop stays open until the push
  lands and mirrors catch up.

---

## How to grow this map

Update `knowledge-map.md` only when:

- an important knowledge domain emerges
- a significant cluster forms (3+ related canonical pages)
- an important synthesis is written
- a major knowledge gap becomes apparent
- the overall knowledge structure materially changes

Do NOT mechanically add every new wiki page.
