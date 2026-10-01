---
type: dashboard
updated: 2026-10-02
---

# Today

## Top priorities

- Ingest today's inbox note (`/kephaleos-ingest` on the "improve Kephaleos top 5" capture) and pick #1 to act on
- Pick first improvement to execute — recommendation stands at **today.md cadence** (this refresh is step 1 of 1 for #1); #2 (decision undercapture) is highest-leverage over 6-month horizon
- If Teams drops after an ISP switch today → `sudo pkill -HUP ZscalerTunnel` (Mode B, see [[Zscaler ZPA route hijack]])

## Active projects

- [[CyberSoft]] — colpaliservice secret-leak hygiene (rewrite prepared, not pushed); sbiqai right-sizing still open from 2026-09-12
- [[Techstyle]] — [[NorthStar]] Apollo Router v2 IaC state captured 2026-09-24, no active work this week
- [[wiki/projects/oikos|Oikos]] — Phase 1 (OPNsense) is the next concrete step; `pve-01` install still open from Phase 8

## Open decisions

- `wiki/decisions/` holds **1 accepted ADR** (`oikos-0001`), **0 proposed**. Three recent judgement calls — ZPA IT-escalation deferral, work-laptop migration timing, vault-PR workflow adoption — are unrecorded. See [[raw/inbox/2026-10-02-0152-kephaleos-improvement-asks]] for the undercapture signal.

## Follow-ups

- From [[work/incidents/2026-09-12-gitlab-runner-cpu-reservation]]: raise `sbiqai` `requests.memory` 256→512Mi · lower `requests.cpu` 250m→50m · reconsider memory-HPA · add `Pending`>5min alert · extend review to `tasktile`/`cybersoft-dtr`/`adminsynonyms`
- From [[work/incidents/2026-09-15-bullzip-sqs-consumer-stall]]: attach DLQ to `auto_bullzip_queue_production` · CloudWatch alarm · fix consumer "task not found" handling · document bypass-task workflow contract · set `SchUseStrongCrypto`/`SystemDefaultTlsVersions` on Windows 2016 hosts
- Colpali: force-push the prepared history-rewrite when a quiet window appears

## Recent important knowledge (last 7 days)

- 2026-10-01 **[[Kubernetes Deployment Readiness Checklist]]** promoted — first entry in `wiki/synthesis/`; 3 learnings + 1 incident + 2 cluster pages compressed into a 6-item pre-deploy lens
- 2026-10-01 [[Zscaler ZPA route hijack]] now covers **two enforcement modes** — Mode A FIB hijack (`route delete/add`) vs Mode B stale NE socket-claim (`pkill -HUP ZscalerTunnel`); incident #4 was the Mode B discovery (Teams signaling, ISP switch)
- 2026-10-01 Two new K8s learnings — [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]] and [[Requests-vs-usage divergence starves cluster capacity]]

## Unresolved problems

- Zscaler ZPA hijack recurring — 4 incidents in ~20 days against unrelated destinations; operator continues local fix over IT scope-review escalation; Mode B's trigger surface (ISP/Wi-Fi switch) is broad enough to keep firing
- Colpali git-history leak — credentials rotated (safety net closed), rewrite prepared but not force-pushed
- `today.md` was 18 days stale before this refresh — dashboard cadence not yet a habit; the vault's "what matters now?" surface decayed silently

## Knowledge gaps

- ZPA policy scope review with IT — under-pressure deferral documented; may need to reopen if Mode B keeps firing on travel/tethering days
- Right-sizing follow-up outcomes on `ccsi-msd-prd` (sbiqai + 3 adjacent workloads)
- Phase II trigger criteria — closed 2026-10-02 by [[wiki/decisions/2026-10-02-phase-ii-triggers]] (ADR #3; first prospective decision in vault)

## Inbox

- 1 item: [[raw/inbox/2026-10-02-0152-kephaleos-improvement-asks]] — "improve Kephaleos, top 5 points" (captured 2026-10-02 01:52)
