---
type: technology
status: active
created: 2026-09-12
updated: 2026-09-12
tags: [kubernetes, eks, aws, ccsi-msd-prd, shared-infra]
maturity: production
---

# ccsi-msd-prd EKS cluster

## What it is

Shared EKS production cluster in AWS `us-west-2`, running Kubernetes
`v1.32.9-eks-ecaa3a6` on Amazon Linux 2023, containerd runtime. Hosts
multiple application namespaces (both prod and UAT) plus the GitLab CI
runner. Not customer-facing directly — it fronts the applications that are.

## Topology (as of 2026-09-12)

Three nodes, all in `us-west-2`:

| Node | Taints | CPU allocatable | Role |
|---|---|---|---|
| `ip-10-51-16-160` | none | ~3900m | General workloads |
| `ip-10-51-26-218` | `observability=true:NoSchedule` | ~8000m | Observability stack (fenced) |
| `ip-10-51-28-62` | none | ~3900m | General workloads |

The observability taint is intentional — general workloads (including CI
runners) cannot land there.

## Workloads

Known namespaces on this cluster:

- `sbiqai` (prod), `sbiqai-uat` — sbiqai deployments + HPAs (min 2, max 5)
- `tasktile` — tasktile deployment + HPA (min 3, max 10)
- `cybersoft-dtr` — cybersoft-dtr deployment + HPA (min 2, max 10)
- `adminsynonyms` — adminsynonyms webapp + HPA (min 1, max 10)
- `colpali` — colpali deployment + HPA (min 1, max 10)
- `gitlab-runner` — GitLab CI runner + ephemeral job pods

## Where we use it

CyberSoft / SBIQ AI production workloads and shared CI runner infrastructure.

## Operational knowledge

- **Systematic CPU over-reservation.** As of 2026-09-12, both schedulable
  nodes sit at 95% CPU *requests* while real usage is 3–5%. The 250m
  request pattern is repeated across `sbiqai`, `tasktile`, `cybersoft-dtr`,
  `adminsynonyms` deployments. This makes the cluster fragile — any pod
  requesting more than ~150m may fail to schedule despite the cluster
  being nearly idle in reality.
- **HPA misconfiguration risk.** Several deployments (notably `sbiqai`)
  scale on memory-utilization with `requests.memory` set below steady-state
  usage, causing HPAs to pin at max. See [[HPA memory-requests pin at max]].
- **No autoscaler.** Cluster Autoscaler / Karpenter is not currently in
  use. Node capacity is fixed; every scheduling failure requires a manual
  intervention.
- **No alerting on `Pending` pods.** A pod stuck in `Pending` will not
  page anyone; the 2026-09-12 GitLab runner incident was reported by a
  developer, not an alert.

## Common issues → solutions

- Pod stuck in `Pending` with `Insufficient cpu` → see [[Incident: GitLab Runner CPU reservation]] for the diagnostic playbook (check HPAs, check requests-vs-actual with `kubectl top`, patch offending HPA `maxReplicas` for quick unblock).

## Related

- [[HPA memory-requests pin at max]]
- [[Incident: GitLab Runner CPU reservation]]
- [[Outage channel announcement runbook]]
