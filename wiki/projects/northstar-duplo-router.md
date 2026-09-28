---
type: project
status: active
created: 2026-09-24
updated: 2026-09-24
aliases:
  - NorthStar
  - northstar-duplo-router
  - Apollo Router (Techstyle)
tags: [project, techstyle, apollo, graphql, kubernetes, eks, terraform, helm]
repo: git@github.com-techstyle:TechstyleOS/northstar-duplo-router.git
confidence: high
---

# Project: NorthStar (northstar-duplo-router)

## Purpose

Infrastructure-as-Code for [[Techstyle]]'s Apollo GraphQL Federation gateway.
Deploys and configures **Apollo Router v2** on self/org-managed EKS across
multiple regions, fronting the federated schema for the Fabletics and
Savage X storefronts.

## Status

Active. `develop` is the integration branch; release branches auto-open
PRs to `develop` and `main`. Apollo Router v2 is current — the legacy
`terraform/apollo/` (v1) module is retained but not modified.

## Architecture (durable knowledge)

- **Terraform** provisions AWS infra (EKS, ALB, Redis, API Gateway, Lambda).
  Active module: `terraform/apollov2/`. Legacy: `terraform/apollo/`.
- **Helm** deploys the Apollo Router container. Chart at `helm/chart/router/`.
- **Rhai scripts** (`helm/chart/router/rhai/`) run as router request/response
  hooks — changes here affect **all** routing behavior across every tenant.
- **k6** load testing for QA and PROD (`k6/`).
- **comparison/** — Node.js tool that diffs DEV vs QA GraphQL responses.

## Tenants / environments

| Tenant | Environment | Region |
|---|---|---|
| `gql-dev01` | Dev | us-west-2 |
| `gql-qa01` | QA | us-west-2 |
| `gql-usw2` | Prod | us-west-2 |
| `gql-use1` | Prod | us-east-1 |
| `gql-euw3` | Prod | eu-west-3 |

On the cluster, tenant `gql-usw2` maps to namespace `duploservices-gql-usw2`
(the `duploservices-` prefix is the legacy naming — see [[Techstyle]]).

## Key files

- `helm/chart/router/values.yaml` — HPA config, resource requests/limits,
  replica counts (parameterized via env vars).
- `helm/chart/router/rhai/main.rhai` — hook logic.
- `terraform/apollov2/asg.tf` — **entirely commented out.** Node capacity
  is not managed from this repo; ASGs and cluster-autoscaler wiring live in
  the platform infra elsewhere.
- `config/{tenant}/apollo.tfvars.json` — per-tenant secrets (gitignored).

## Operational knowledge

- **Router pod requests are heavily oversized** (as of 2026-09-23):
  `router` container requests **1 CPU / 2 GiB memory** but real usage sits
  at ~50m CPU / a few hundred MiB per pod. On `gql-usw2` this made the
  cluster 97% CPU-*requested* while nodes were 7–11% actually utilized.
  See [[Requests-vs-usage divergence starves cluster capacity]] (proposed
  synthesis).
- **HPA is per-tenant router**, min 6 / max 10 in `gql-usw2`, targeting
  60% CPU / 75% memory. Never scales up in practice because oversized
  requests keep the metric near zero.
- **Node capacity is fixed** — cluster autoscaler on `duplo-prd-usw2`
  is running but managing zero node groups (see the cluster page below).

## Lessons learned

- [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]]

## Related

- [[Techstyle]]
- [[duplo-prd-usw2 EKS cluster]]
