---
type: dashboard
created: 2026-09-13
updated: 2026-10-08
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
  Backed by **five** lived incidents in ~26 days (FortiClient IPsec,
  partner SFTP TCP RST, AWS EC2 CI runner SSH silent timeout, MS Teams
  signaling after ISP switch, AWS NLB in front of cybersoft pgbouncer).
  Covers **two distinct enforcement layers**: Mode A (FIB route hijack
  — fix with `route delete/add`) and Mode B (stale NetworkExtension
  claim after network context change — fix with
  `pkill -HUP ZscalerTunnel`). Incident #5 added a **persistent-override
  LaunchDaemon** (§Fix §4) as a middle-path workaround and surfaced the
  **two-egress-IP side-gotcha** (direct ISP vs Zscaler-tunneled) that
  breaks naive AWS SG whitelisting. Third-occurrence rule triggered →
  escalation is now a ZPA policy scope review, not per-host bypass tickets.
- [[wiki/runbooks/macos-vpn-ne-triage|Runbook: macOS VPN/NE triage]] —
  10-second decision-tree extraction of the above solution page for
  mid-incident use. Pairs the two diagnostic measurements (`nc -zv`
  shape + `route get` interface) into a fingerprint table that routes
  to Mode A / Mode B / "not this runbook" in one glance.
- [[wiki/decisions/2026-10-01-zpa-escalation-deferral]] — formal ADR
  behind why the operator keeps running the runbook rather than
  escalating; five explicit revisit triggers.

### Incident communication

- [[Outage channel announcement runbook]] — CyberSoft `! IMP - Outages`
  channel format for live incident + inline Sev-3 postmortem
- [[Ops communication templates]] — portable maintenance / change /
  outage / postmortem templates with connected worked samples

### Kephaleos itself

- `CLAUDE.md` — Phase 1.1 knowledge-maintainer charter
- [[Learning: Kephaleos must be accessible from any working directory]]
- [[wiki/decisions/2026-10-02-phase-ii-triggers]] — ladder for allowing any charter-§28-forbidden feature (embeddings, RAG, external ingestion, local models, etc.) with concrete measurable triggers and cross-phase hard limits

### Networking concepts (new 2026-10-02)

First two canonical concept pages — seeded to deduplicate the inline restatements that had grown across the ZPA synthesis:

- [[wiki/concepts/cgnat-100.64.0.0-10|CGNAT (100.64.0.0/10)]] — RFC 6598 Shared Address Space; the fingerprint IP range used by ZCC/WARP/Tailscale/Twingate for their local `utun` addresses. Why a `100.64.*` entry in `route get` is near-pathognomonic for a client-side tunnel on a workstation.
- [[wiki/concepts/macos-network-extension|macOS NetworkExtension]] — Apple's user-space framework for `NEPacketTunnelProvider`. Explains the two-stage connect() path (NE `includedRoutes` before FIB) that makes `route get` lie, and why the Mode B stale-NE failure in [[Zscaler ZPA route hijack]] is a structural consequence of the framework's design, not a Zscaler bug per se.

Both concept pages are **referenced from** the solution/runbook/decision triplet (deduplication landed inline on each page), not standalone — the vault's design preference per charter §19 is compression into fewer, cross-linked pages, not more pages.

---

## Important syntheses

- **Zscaler ZPA hijack pattern (5 incidents → one canonical solution, two
  modes, one runbook, one decision, one persistent-workaround daemon).**
  Same machine, five unrelated destinations in ~26 days: corp FortiGate
  IPsec (UDP timeout), partner SFTP (fast TCP RST), employer EC2 CI
  runner SSH (silent timeout + ICMP), MS Teams signaling after ISP
  switch (`EADDRNOTAVAIL` in <100 ms), AWS NLB fronting cybersoft
  pgbouncer (fast RST → silent timeout toggle within one destination).
  Fingerprints: **Mode A (FIB hijack)** → `route -n get <ip>` shows
  CGNAT-range next-hop (100.64.0.0/10); fix with route override.
  **Mode B (stale NE)** → `route` is clean but socket bind returns
  `EADDRNOTAVAIL` immediately; fix with `pkill -HUP ZscalerTunnel`.
  GUI toggles for ZIA/ZPA do not clear Mode B — the stale claim lives
  in the system LaunchDaemon layer. The synthesis now spans three
  pages plus a staged persistent workaround:
  [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]]
  (deep root-cause + policy-side fix + §4 LaunchDaemon workaround + two-egress-IP side-gotcha),
  [[wiki/runbooks/macos-vpn-ne-triage]] (10-second triage reflex),
  [[wiki/decisions/2026-10-01-zpa-escalation-deferral]] (why we run the
  runbook rather than escalate; 2026-10-08 status check added after #5).
  First deep→runbook→decision triplet in the vault — pattern worth
  repeating when a solution gets used in anger more than twice.
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
- **Phase II trigger criteria — formally defined 2026-10-02.** The gap
  "when does Phase II start?" is closed by
  [[wiki/decisions/2026-10-02-phase-ii-triggers]] — a prospective ADR
  defining concrete entry triggers (inbox-age, recall-miss rate, canonical
  page count, manual-sync friction, LLM cost, retrieval latency) for each
  class of Phase 1.1-forbidden feature, their adoption order within Phase
  II (embeddings first, autonomous agents last / Phase III only), and
  the cross-phase hard limits (portability, no secrets, no auto-overwrite,
  no autonomous `wiki/` writes). No feature adopted today — this is the
  ladder, not the step.

---

## How to grow this map

Update `knowledge-map.md` only when:

- an important knowledge domain emerges
- a significant cluster forms (3+ related canonical pages)
- an important synthesis is written
- a major knowledge gap becomes apparent
- the overall knowledge structure materially changes

Do NOT mechanically add every new wiki page.
