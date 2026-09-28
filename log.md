# Kephaleos — Activity Log

Append-only. One line per meaningful event. Newest at the bottom.

Format: `YYYY-MM-DD <op>: <one-line description> [→ [[page]]]`

Ops (Phase 1.1): `remember` (only for canonical creation/merge/major update),
`decision`, `ingest`, `synthesize`, `restructure`, `resolve-conflict`,
`promote`, `archive`. Per charter §20 do NOT log every recall/search/read.

---

2026-09-12 init: Kephaleos Phase I foundation created — architecture, CLAUDE.md, templates, global commands.
2026-09-12 remember: Kephaleos must be accessible from any working directory (E2E bootstrap test) → [[wiki/learnings/2026-09-12-kephaleos-global-access]]
2026-09-12 recall: What did I learn about global Kephaleos access? → [[wiki/learnings/2026-09-12-kephaleos-global-access]]
2026-09-12 remember: VPN client fails when another VPN/ZTNA agent (Zscaler ZPA) hijacks the route to the gateway → [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]]
2026-09-12 remember: GitLab runner scheduling failure on ccsi-msd-prd — sbiqai HPAs pinned at max due to undersized requests.memory; wasted CPU reservation blocked runner → [[work/incidents/2026-09-12-gitlab-runner-cpu-reservation]]
2026-09-12 remember: ccsi-msd-prd EKS cluster topology, workloads, and systemic CPU over-reservation → [[wiki/technologies/ccsi-msd-prd-eks-cluster]]
2026-09-12 remember: Memory-based HPA silently pins at max when requests.memory is set below steady-state usage — generalized learning → [[wiki/learnings/2026-09-12-hpa-memory-requests-pin-max]]
2026-09-12 remember: Internal outage-channel announcement runbook (! IMP - Outages format + combined post with inline postmortem via <br>) → [[wiki/runbooks/outage-channel-announcement]]
2026-09-13 restructure: Phase 1.1 knowledge-intelligence upgrade — charter rewritten (write policy, canonical identity + aliases, project context, provenance/claim-level uncertainty, temporal knowledge, recall ranking + response format, logging discipline, lint upgrade); remember/recall/review/lint/decision commands rewritten; knowledge-map.md and today.md added; index.md and home.md trimmed; aliases added to existing canonical pages. Nothing deleted.
2026-09-13 remember: k8s-gitlab-runner project — architecture, IRSA, DinD tradeoffs, node-pinning gap in prod values, cleanup pass (moved trust policy to docs/, dropped unused Dockerfile, README rewrite) → [[wiki/projects/k8s-gitlab-runner]]
2026-09-13 remember: GitLab Runner token rotation runbook — canonical procedure for rotating exposed/compromised `glrt-` tokens on the shared runner → [[wiki/runbooks/gitlab-runner-token-rotation]]
2026-09-13 remember: aws-tf-network project — single-VPC (10.51.0.0/16, us-west-2) with 4 envs as subnet tiers, VPN + 4 peerings, TF 0.14.9/AWS ~3.27; 8 non-obvious known issues (AZ region literal, empty VPC endpoints, hardcoded pcx IDs, dev VPN commented, dev/staging CIDR overlap, README/TF drift, CI stub, no lint); docs overhaul (network.md + Mermaid diagrams, cidr-allocation, onboarding, known-issues) landed in-repo same day → [[wiki/projects/aws-tf-network]]
2026-09-13 remember: Portable ops communication templates — maintenance announcement + change management + outage/incident + postmortem, with connected worked samples (PG 14→15 pgbouncer scram cutover failure); cross-linked to existing cybersoft-specific outage-channel runbook → [[wiki/runbooks/ops-communication-templates]]

2026-09-14 capture: this settings always have change update on all alb → raw/inbox/2026-09-14-1518-alb-listener-drift.md
2026-09-14 capture: taskmanager-03 vs taskmanager (safe to decom, empty since 2024-08 → raw/inbox/2026-09-14-1526-taskmanager-03-decom.md
2026-09-14 capture: alb-prod-unstractlte listener destroy blocked (in-use by NLB) → raw/inbox/2026-09-14-1548-alb-destroy-listener-inuse.md
2026-09-14 capture: decommission alb-tblextraction (NLB-first destroy order) → raw/inbox/2026-09-14-1550-decom-alb-tblextraction.md
2026-09-14 remember: created wiki/technologies/aws-elbv2-alb-nlb.md (ALB/NLB operational patterns) + improved wiki/projects/aws-tf-network.md (tag ownership map, ghost stacks, TLS remediation history)
2026-09-14 capture: windows 2016 blocks TLS 1.3 upgrade plan for some services → raw/inbox/2026-09-14-2013-windows-2016-tls.md
2026-09-15 remember: incident work/incidents/2026-09-15-bullzip-sqs-consumer-stall.md + created wiki/technologies/dotnet-framework-tls.md + updated aws-elbv2-alb-nlb.md and aws-tf-network.md cross-refs
2026-09-15 remember: second confirmed ZPA route hijack (partner SFTP endpoint, TCP RST symptom vs prior IKE timeout) — strengthens [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] with new "timeout vs refused" fingerprint + recurring-problem escalation note → [[raw/notes/2026-09-15-zpa-hijack-partner-sftp]]
2026-09-15 remember: created [[wiki/organizations/cybersoft]] (org page with 6-service catalog, offices, delivery platforms) and [[wiki/projects/cybersoftbpo-website]] (Next.js + Payload rebuild of cybersoftbpo.com — decisions, extracted design tokens, gotchas, workflow scripts); back-linked from [[wiki/projects/cybersoft-sftp]]. First `organizations/` canonical page in the vault.
2026-09-17 remember: created [[wiki/projects/yakap]] (Rails 8.1 app under Ttsi service line — architecture, environments, TLS strategy, 2026-09-17 HTTP-01→DNS-01 migration lesson); back-linked from [[wiki/organizations/cybersoft]]. Durable pattern captured: when :80 becomes unreachable on cybersoft-tenancy hosts, switch `certbot_method` to `dns-route53` — the ansible `certbot` role already supports it.
2026-09-18 remember: created [[wiki/projects/oikos]] (Oikos homelab canonical project page) + full operational tree at [[work/projects/oikos/README]] — docs scaffold, 13-phase roadmap redesigned by dependency, address plan (172.27.0.0/16, 9 VLANs), naming conventions, T14 bootstrap node page, CHANGELOG. Homelab repo at `~/workspace/personal/homelab/` scaffolded with CLAUDE.md pointing to Kephaleos.
2026-09-18 decision: ADR oikos-0001 accepted — internal DNS zone is `oikos.home.arpa` (RFC 8375-compliant subdomain of `home.arpa`); rejects bare `.arpa` labels and un-suffixed `home.arpa` → [[wiki/decisions/oikos-0001-dns-namespace]]
2026-09-18 restructure: Oikos phase plan renumbered — Packet Tracer moved to parallel [[work/projects/oikos/study/README|study track]] (not a phase); Phase 1 is now OPNsense/firewall (was Phase 2); downstream phases shifted by −1; total numbered phases 0–11 (was 0–12). Aligns with operator mental model that Phase 1 = first real infra change.
2026-09-18 remember: drafted 11 phase pages (Phase 1 OPNsense through Phase 11 Automation) + consolidated hardware roadmap for Oikos under [[work/projects/oikos/roadmap]] — each phase covers objective/why-now/prereqs/hardware/software/network/implementation/CCNA/labs/break-fix/verification/security/monitoring/docs/diagrams/completion-checklist. Templates enforce discipline; break-fix drills baked into every phase.
2026-09-22 remember: third confirmed Zscaler ZPA route hijack — AWS EC2 CI runner over SSH 22 (`us-west-2`), silent timeout shape. Third-occurrence rule triggered: canonical solution page updated to escalate from per-host bypass tickets to a ZPA policy scope review with IT; new stacking-failure-mode note added for employer-owned SG-restricted targets → [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] + [[raw/notes/2026-09-22-zpa-hijack-ec2-runner-ssh]]
2026-09-24 remember: created [[wiki/organizations/techstyle]] (Techstyle Fashion Group / TechstyleOS — Fabletics, Savage X) + [[wiki/projects/northstar-duplo-router]] (Apollo Router v2 IaC for the federated storefront gateway across us-west-2/us-east-1/eu-west-3) + [[wiki/technologies/duplo-prd-usw2-eks-cluster]] (`duploinfra-api-us-west-2` prod EKS topology). First Techstyle canonical pages.
2026-09-24 remember: colpaliservice project — FastAPI + fastembed + PDF pipeline in data-engineering group, deployed to `colpali` ns on ccsi-msd-prd EKS via SHA-pinned `deploy.sh`; documents architecture, deploy sequence, double-hidden OpenAPI docs, and historical ConfigMap secret leak (AWS key + DB password) removed on `chore/jhonatsz` but still in git history → [[wiki/projects/colpaliservice]]
