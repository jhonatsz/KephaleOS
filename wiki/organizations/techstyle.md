---
type: organization
status: active
created: 2026-09-24
updated: 2026-09-24
aliases:
  - TechStyle
  - TechstyleOS
  - Techstyle Fashion Group
tags: [organization, fashion, ecommerce, employer]
confidence: high
---

# Techstyle

Fashion e-commerce group. GitHub org handle: `TechstyleOS`.

## Context

- **Kind:** Consumer fashion e-commerce (private company)
- **Brands (as observed in the NorthStar routing layer):** Fabletics (FL),
  Savage X (SX)

## Architecture surfaces I've touched

- **Apollo GraphQL Federation** as the customer-facing composition layer
  for the storefronts, deployed on self/org-managed EKS in AWS.
  See [[northstar-duplo-router]].
- **Multi-region prod EKS** — `us-west-2`, `us-east-1`, `eu-west-3` —
  plus `us-west-2` dev and QA. All fronted by Apollo Router v2.

## Naming legacy

Cluster and namespace names still carry `duplo-` / `duploservices-` prefixes.
The clusters are **self/org-managed EKS**, not DuploCloud-platform-managed —
the migration off DuploCloud already happened; the naming didn't get
refactored. Do not assume DuploCloud (the third-party platform) gates
cluster-level configuration (IAM/RBAC access entries, console visibility,
etc.). That's fully in-house.

## Systems and projects at Techstyle

- **[[northstar-duplo-router]]** — Apollo Router v2 IaC (Terraform + Helm)
  running the federated GraphQL gateway on EKS
- **[[duplo-prd-usw2 EKS cluster]]** — prod EKS cluster in `us-west-2`
  hosting the Apollo Router for the US-West federation region

## Related

- [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]]
