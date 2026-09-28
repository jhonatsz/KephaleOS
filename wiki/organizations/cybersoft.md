---
type: organization
status: active
created: 2026-09-15
updated: 2026-09-24
aliases:
  - CyberSoft
  - CyberSoft BPO
  - CyberSoft Content Services
  - Cybersoft Content Services, Inc.
tags: [organization, bpo, cybersoft, service-catalog]
confidence: high
---

# CyberSoft

Business process outsourcing organization with continuity-and-resilience
positioning across regulated industries (mortgage, healthcare, legal,
records, back office). Formally *Cybersoft Content Services, Inc.*
Public domain `cybersoftbpo.com`.

## Context

- **Kind:** BPO / managed services (private company)
- **Positioning:** "Continuity and resilience amidst disruption" —
  disaster-tolerant remote-first delivery model
- **Website:** cybersoftbpo.com (Wix, being replaced — see
  [[wiki/projects/cybersoftbpo-website]])
- **Contact email of record:** info@cybersoftbpo.com

## Offices (as of 2026-09-15)

Sourced from the live site footer during the 2026-09-15 scrape.

| Location | Address | Phone |
|---|---|---|
| San Francisco | 100 Pine St. (Suite 1250), San Francisco, California 94111 | +1 (415) 745-3034 |
| Quezon City — Pacific Corporate Center | 7th Floor, 131 West Ave, Quezon City, PH 1100 | +63 (2) 8-374-6633 |
| Quezon City — Centris Cyberpod 3 | 21st Floor, EDSA corner Quezon Avenue, Quezon City, PH 1100 | +63 (2) 8-374-6634 |

> [!info] Time-sensitive
> Office details are copied from the current public site. Verify with
> internal directory before external use.

## Service catalog

Six top-level service lines. Descriptions below are the current public
positioning from cybersoftbpo.com and should be replaced with internal
authoritative wording when captured.

### Mortgage Support Services

Mortgage operations continuity — remote/at-home delivery model designed
to keep processing going through disruption. Public tagline: *"Your
mortgage operations can continue amidst disaster. Work at home or
anywhere with our best platforms yet."*

### Content and Records Management

Records digitization and data extraction, delivered via AI-assisted
production models. Public tagline: *"From digitization of old records
to extraction of data — accomplished through AI-assisted production
models."*

### Healthcare Support Services

Combined platform + knowledge-worker delivery for healthcare admin/ops
support. Public tagline: *"Exceptional service support systems through
a combination of platforms and knowledge workers."*

### Back Office Support Services

Horizontal BPO — customer support, sales, accounting, lead generation,
research. Public tagline: *"From customer support, to sales, to
accounting, to lead generation, to research — you can outsource to
us."*

### Legal and Medical Transcription

Compliance-aware transcription in the medium the customer needs.
Public tagline: *"Have your requirements for transcription delivered
in time observing all compliance guidelines, in the medium you need."*

### Business Geographics Services

Customized navigation and tracking. Public tagline: *"Customized
navigation and tracking services for companies and organizations with
special needs."*

## Delivery platforms

Named service delivery platforms surfaced during the live-site scrape
(2026-09-15). Enumerated from platform logo assets in the site's
media library — not from an internal platform inventory. Follow-up:
map each platform to the service line(s) it supports.

- **DataRaQ**
- **eClaims** (branded "eClaims by [partner]")
- **medixHIS** — health information system
- **medixPASS**
- **medixWATCH**
- **Ttsi** — "Your Health Compliance Innovator"
- **SBIQ** (referenced by logo asset `sbiq_20logo_20new_20angle_20rgb-02.png`)

> [!warning] Hypothesis
> The `medix*` cluster (HIS, PASS, WATCH) plausibly belongs to the
> Healthcare Support Services line; Ttsi likely also healthcare-adjacent
> given its tagline. Not yet confirmed against internal documentation.

## Compliance & policy documents

Referenced by the public site navigation (public copies to be replaced
by internal authoritative versions when captured):

- Privacy Policy
- Service Legal Agreement
- Information Security Program
- Audit Checklist

## Systems and projects at CyberSoft

- **[[wiki/projects/cybersoft-sftp]]** — partner-facing SFTP endpoint
  (`sftp.cybersoftbpo.com:2233`), ~78 accounts, Ubuntu + OpenSSH +
  bindfs, migration to 26.04 planned
- **[[wiki/projects/cybersoftbpo-website]]** — Next.js + Payload CMS
  rebuild of the public website (replacing Wix), in progress
- **[[wiki/projects/yakap]]** — Rails 8.1 application under the Ttsi
  service line; staging + 5-VM production stack, provisioned by the
  shared `devops/ansible` repo
- **[[ccsi-msd-prd EKS cluster]]** — shared prod EKS cluster referenced
  from the ops incident history
- **[[wiki/projects/colpaliservice]]** — FastAPI embeddings + PDF pipeline
  in the `data-engineering` GitLab group, deployed to the `colpali`
  namespace on the shared EKS cluster
- **[[wiki/runbooks/outage-channel-announcement]]** — internal outage
  communication runbook

## Related

- [[wiki/runbooks/gitlab-runner-token-rotation]]

## Sources

- Live-site scrape 2026-09-15 (see
  [[wiki/projects/cybersoftbpo-website]] for the extractor and content
  dump)
- [[wiki/projects/cybersoft-sftp]] for infra ownership context
