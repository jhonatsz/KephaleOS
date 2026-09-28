---
type: technology
status: active
created: 2026-09-24
updated: 2026-09-24
aliases:
  - "duplo-prd-usw2 EKS cluster"
  - "duplo-prd-usw2"
  - "duploinfra-api-us-west-2"
tags: [kubernetes, eks, aws, techstyle, duplo-prd-usw2, apollo-router]
maturity: production
time-sensitive: true
---

# duplo-prd-usw2 EKS cluster

## What it is

Techstyle's production EKS cluster in AWS `us-west-2`, cluster name
`duploinfra-api-us-west-2`. Hosts the US-West production Apollo Router
tenant (`gql-usw2`) plus pricing services (`fbl-pricing-service-prod-usw2`,
`sxf-pricing-service-prod-usw2`) and other prod workloads. Self/org-managed
EKS — the `duplo-` prefix is legacy naming (see [[Techstyle]]).

## Topology (as of 2026-09-23)

Three nodes, ~5 days old, Amazon Linux 2023, containerd:

| Zone | CPU allocatable | Actual CPU % | CPU requested % |
|---|---|---|---|
| us-west-2a | ~3900m | ~7% | ~97% |
| us-west-2b | ~3900m | ~11% | ~97% |
| us-west-2c | ~3900m | ~7% | ~97% |

Real utilization is near-idle; scheduler view of the cluster is nearly full.

## Where we use it

Techstyle production for the US-West-2 Apollo Router federation region
and adjacent services. See [[northstar-duplo-router]].

## Operational knowledge

- **Cluster Autoscaler is deployed but managing zero node groups.**
  Discovered 2026-09-23. `cluster-autoscaler` in `kube-system` is
  configured with
  `--node-group-auto-discovery=asg:tag=duplocloud.com/eks-cluster-autoscaler/enabled=true,duplocloud.com/eks-cluster-autoscaler/cluster=$(CLUSTER_NAME)`,
  but **no ASG in the account carries those tags for this cluster**.
  Consequence: CA logs `Node X should not be processed by cluster autoscaler
  (no node group config)` for every node, and pods that can't schedule
  emit `NotTriggerScaleUp` events with **an empty reason** (blank after
  the colon). Fix path is AWS-side — tag the node group ASG(s) with both
  the `enabled=true` and `cluster=duploinfra-api-us-west-2` tags. See
  [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]].
- **Systematic CPU over-reservation on the Apollo Router.** Router pods
  request 1 full CPU each but use ~50m. HPA min-replicas alone (6) reserves
  6 CPU of ~11.7 allocatable; rolling deploys push it over. Nodes sit
  7–11% actually utilized while scheduler reports 97% requested.
- **Long-tail Pending pods (333 days).** As of the scan: `cadvisor-*`,
  `filebeat-*`, `node-exporter-*`, `otel-collector-*`, `redis-cli-*` in
  `duploservices-gql-usw2` have been `Pending` for 333 days. Same
  root cause (no autoscaling to satisfy them; CPU-request budget
  exhausted). Priority class ≤ -10, so CA marks them expendable and
  ignores them for scale-up — they will not self-heal even after CA
  is fixed unless requests are reduced.
- **Deploy rollouts routinely surge past scheduler capacity.** Because
  the HPA sits at min (6) and `maxSurge` pushes to 8 during a rolling
  update, and nodes are already at 97% requested, every deploy manifests
  the same `FailedScheduling: Insufficient cpu` symptom transiently.

## Common issues → solutions

- Pod stuck in `Pending` with `Insufficient cpu` and CA event
  `NotTriggerScaleUp` reason blank → see
  [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]].
- Rollout stuck mid-deploy → temporary unblock: bump the node group ASG
  `DesiredCapacity` by 1 manually; permanent fix: right-size the router
  `resources.requests.cpu` in `helm/chart/router/values.yaml`.

## Related

- [[Techstyle]]
- [[northstar-duplo-router]]
- [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]]
- [[HPA memory-requests pin at max]] — same *class* of pathology on a
  different cluster ([[ccsi-msd-prd EKS cluster]]): request-vs-usage
  divergence starves the scheduler's reservation budget while nodes idle.
