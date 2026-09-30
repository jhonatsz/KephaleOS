---
type: learning
status: active
created: 2026-09-12
updated: 2026-10-01
aliases:
  - "HPA memory-requests pin at max"
  - "Memory HPA silent CPU reservation exhaustion"
tags: [kubernetes, hpa, autoscaling, resource-requests, capacity-planning]
confidence: high
---

# Undersized requests.memory causes memory-based HPAs to pin at max — silently starving cluster CPU reservation

## Context

Learned during the 2026-09-12 incident where GitLab CI runner pods on
`ccsi-msd-prd` could not be scheduled. Cluster CPU was nearly idle
(3–5% real usage) but 95% *reserved* on the two schedulable nodes.

## What happened

Two sbiqai HPAs (`sbiqai/sbiqai-hpa`, `sbiqai-uat/sbiqai-hpa`), both
configured with `memory averageUtilization: 80`, were pinned at
`maxReplicas: 5`. CPU utilization on all 10 sbiqai pods was ~0%.
The HPAs were firing on memory not because the workload was under
memory pressure, but because `requests.memory` (256Mi) was set below
the steady-state memory footprint (~200Mi/pod). 200 / 256 ≈ 78% —
sitting at the 80% target line permanently.

Because each pod also carried `requests.cpu: 250m` while using ~1m,
10 pinned pods reserved 2500m of CPU across a cluster with only ~7800m
schedulable — blocking new pods (including CI runners) from scheduling.

## The lesson

**When configuring a memory-based HPA, `requests.memory` must be set
comfortably above the workload's steady-state memory footprint** — or
the HPA will scale purely because "resting memory" looks like "load."
The HPA metric is `usage / request`, not `usage / limit`, so a too-tight
request makes the workload look permanently overloaded.

Corollary: memory-based HPA is a poor fit for **steady-state workloads**
(loaded models, warm caches, JVMs). Their memory footprint is close to
constant regardless of load. Prefer CPU-based HPA, or a custom metric
tied to actual request rate, for these workloads.

Corollary: the failure mode is silent. Everything looks healthy — pods
running, HPAs at max, alerts quiet — right up until something else on
the cluster can't schedule because the reservation budget is gone.

## Why it matters

This pattern will recur wherever:

- HPA uses memory-utilization
- `requests.memory` is set close to real steady-state usage
- The workload's memory is not load-driven

Every cluster with tightly-sized memory requests + memory-based HPA is
at risk. The symptom is *another* pod failing to schedule, not the
misbehaving deployment itself — which makes diagnosis unintuitive.

## Detection

`kubectl get hpa -A` — any HPA showing `REPLICAS == MAXPODS` with
`memory` well above the target and CPU near zero is a likely instance
of this pattern. Cross-reference with `kubectl top pods` (real usage)
and the deployment's `requests.memory`.

## Related

- [[Incident: GitLab Runner CPU reservation]]
- [[ccsi-msd-prd EKS cluster]]
- [[Kubernetes Deployment Readiness Checklist]] — synthesis (item 2)
  covers memory-based HPA on steady-state workloads
