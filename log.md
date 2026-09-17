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
