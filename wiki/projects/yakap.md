---
type: project
status: active
created: 2026-09-17
updated: 2026-09-17
aliases:
  - Yakap
  - phic_yakap
  - yakap-staging
tags: [project, cybersoft, ttsi, rails, ansible, minio, letsencrypt]
repo: git@git.cybersoftbpo.com:devops/ansible.git
---

# Project: Yakap

Rails 8.1 application delivered under the **Ttsi** service line at
[[CyberSoft]]. Provisioned by the shared `devops/ansible` repo.
Public hostname: `yakap-staging.ttsibpo.com` (staging).

## Purpose

Rails web application (name and business scope not yet captured here —
follow-up: fill in from product docs). Uses PostgreSQL 16 for app data
plus three separate Solid databases (cache, queue, cable) — see the
Solid-gems ADR. Uses MinIO for Active Storage.

## Environments

| Env | Shape | Public hostname |
|-----|-------|-----------------|
| staging | single VM — nginx + Puma + Solid Queue + PG16 + pg-backup, co-located | `yakap-staging.ttsibpo.com` |
| staging (storage) | dedicated VM — MinIO behind nginx (S3 API + console on split hostnames), internal-only IP | `yakap-minio-staging.ttsibpo.com`, `yakap-minio-api-staging.ttsibpo.com` |
| production | 5-VM stack — HAProxy LB + app VMs + dedicated DB + dedicated MinIO | (see uk_ie / yakap_lb inventories) |

## Architecture (durable knowledge)

Only the parts that would be painful to rediscover. Detail lives in the
`devops/ansible` repo and its `docs/architecture.md`.

- **Nginx** terminates TLS; proxies `127.0.0.1:3000` (Puma).
- **Puma** runs under a dedicated `yakap` user via systemd
  (`puma-yakap`); 4 workers × 5 threads.
- **Solid Queue** runs as its own systemd service (`solid-queue-yakap`),
  not embedded in Puma (see ADR-015).
- **PostgreSQL 16** with four databases — `yakap`, `yakap_cache`,
  `yakap_queue`, `yakap_cable` (see ADR-007).
- **Deploy path**: Ansible provisions infrastructure once, Capistrano
  handles every release; Ansible can invoke Capistrano from CI (ADR-004,
  ADR-010).
- **TLS**: Let's Encrypt via the shared `certbot` role. Method varies
  per host — see TLS section below.
- **App code lives at** `/var/www/yakap/current/` (Capistrano release
  symlink layout).

## TLS strategy

Two validation methods coexist in this stack, chosen per host:

| Host | Method | Why |
|------|--------|-----|
| yakap-staging (app) | **DNS-01 via Route53** (as of 2026-09-17) | Port 80 exposure at the tenancy is unreliable — see [Operational history](#operational-history) |
| yakap-minio-staging | DNS-01 via Route53 | Host is on a private IP with no public reachability (ADR-016, ADR-017) |
| yakap_lb (production) | http-01-webroot | HAProxy stays up during renewal; certbot serves the challenge from a webroot mounted behind HAProxy |
| yakap_minio (production) | DNS-01 via Route53 | Same as staging |
| uk_ie (production) | DNS-01 via Route53 | Same pattern applied to the UK/IE shared-tenancy nginx host |

Pattern: **when a host cannot reliably expose :80 to Let's Encrypt,
switch to DNS-01 via Route53.** The ansible `certbot` role gates on
`certbot_method` in group_vars, so the switch is a config-only change
plus a one-time cert delete on the box. See ADR-017.

## Operational history

### 2026-09-17 — Staging cert expired; migrated yakap-staging to DNS-01

**Symptom.** `yakap-staging.ttsibpo.com` served an expired cert
(expired 2026-09-15). `certbot.timer` was healthy and firing twice
daily since at least Sep 13, but every renewal attempt failed with:

> `Timeout during connect (likely firewall problem)`
> during HTTP-01 challenge fetch of
> `http://yakap-staging.ttsibpo.com/.well-known/acme-challenge/...`

Let's Encrypt reached the correct public IP but the TCP connect to :80
timed out (packets dropped, not RST) — the firewall / edge / security
group between the internet and the VM had stopped allowing inbound :80
at some point between the last successful renewal and Sep 13.

**Fix.** Migrated yakap-staging from HTTP-01 (`--nginx` plugin) to
DNS-01 via Route53, mirroring the pattern already used by
`yakap_minio`, `yakap_lb`, and `uk_ie`. Two-file change:

- Slim `inventories/staging/group_vars/all/certbot.yml` back to shared
  defaults only (`certbot_method: http-01`, empty domains, shared
  email).
- New `inventories/staging/group_vars/yakap_app.yml` with
  `certbot_method: dns-route53`, the yakap-staging domain, and the
  existing `vault_route53_*` credentials.

Then on the VM: `certbot delete --cert-name yakap-staging.ttsibpo.com`
(needed because the old renewal conf was pinned to
`authenticator = nginx`), then re-run the `certbot` tag of
`setup-yakap-service.yml`. Fresh cert issued via DNS-01; auto-renewal
uses the same plugin going forward — no more port-80 dependency.

**Note on the gotcha during rollout.** Running the playbook before
editing the group_vars still hit the http-01 branch (which is the
`all/certbot.yml` default). Because the earlier `certbot delete` had
already removed `fullchain.pem`, the `--nginx` plugin failed
immediately on `nginx -t` (nginx config still referenced the removed
cert). The takeaway: **flip `certbot_method` in group_vars BEFORE
deleting the old cert.**

**Lesson (durable).** When HTTP-01 breaks silently on a CyberSoft
tenancy host, don't fight the firewall — switch to DNS-01. The repo
already supports it; the switch is minutes not hours. Systemd's
`certbot.timer` will silently retry a broken renewal for weeks; the
first user-visible signal is an expired cert. Consider adding an alert
on `certbot renew --dry-run` exit status if this pattern repeats.

## Key decisions

Decisions live as ADRs in `devops/ansible/docs/adr/`. Notable ones
touching Yakap:

- ADR-004 — Ansible triggers Capistrano
- ADR-006 — Dedicated `yakap` user for deploy
- ADR-007 — Separate databases for Solid gems
- ADR-008 — Let's Encrypt SSL with auto-renewal (HTTP-01 default)
- ADR-013 — Separate staging Rails environment
- ADR-014 — MinIO for Active Storage
- ADR-015 — Solid Queue as dedicated systemd service
- ADR-016 — MinIO on dedicated VM with split console/API hostnames
- ADR-017 — Let's Encrypt DNS-01 via Route53 for internal services
- ADR-018 — Pin MinIO to pre-rebrand AGPL release
- ADR-019 — Auto-restart Rails services on `.env` change

## Related

- [[CyberSoft]] — parent organization; Yakap sits under the Ttsi
  service line
- [[wiki/runbooks/ops-communication-templates]] — maintenance
  announcement format used for the 2026-09-17 corrective work
  (planned window under our control → maintenance, not outage)

## Sources

- `devops/ansible` — `docs/architecture.md`, `docs/adr/017-*`,
  `roles/certbot/*`, `inventories/staging/group_vars/*`
- Direct experience — 2026-09-17 cert migration
