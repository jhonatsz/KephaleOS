---
type: incident
status: mitigated
created: 2026-09-12
updated: 2026-09-13
aliases:
  - "Incident: GitLab Runner CPU reservation"
  - "GitLab CI runner Pending pods 2026-09-12"
tags: [kubernetes, eks, hpa, gitlab-ci, sbiqai, ccsi-msd-prd]
severity: SEV3
duration: unknown
---

# Incident: GitLab CI runner pods failing to schedule on ccsi-msd-prd — 2026-09-12

## Summary

GitLab CI runner pods on the shared EKS cluster `ccsi-msd-prd` (us-west-2)
could not be scheduled. Scheduler reported `Insufficient cpu` on the two
schedulable nodes and untolerated taint on the third. Root cause was
**CPU reservation saturation, not real CPU load** — the sbiqai prod + UAT
HPAs were pinned at their max replicas, and each pod reserved ~60× more
CPU than it used.

## Timeline

- HH:MM — CI pipeline on GitLab project 618 reports runner pod stuck in `Pending`
- HH:MM — Cluster scan confirms 95% CPU reservation on both schedulable nodes; sbiqai HPAs identified as the reservation source
- HH:MM — `sbiqai-uat` HPA `maxReplicas` patched 5 → 3 (frees ~500m CPU reservation)
- HH:MM — Runner pod schedules; pipeline resumes
- HH:MM — Status marked RESTORED

## Impact

- New CI jobs across all repos using the shared runner unable to start
- Running CI jobs, production application traffic, and end-user services **unaffected**
- Blast radius limited to CI throughput

## Root cause

Two-part misconfiguration compounded:

1. **HPA memory metric firing at steady state.** Both `sbiqai` and `sbiqai-uat`
   deployments had `requests.memory: 256Mi`, but real usage was ~200Mi/pod.
   HPA target `memory averageUtilization: 80` compares real usage to *request*,
   so utilization sat at ~78% permanently, pinning both HPAs at `maxReplicas: 5`
   (10 sbiqai pods total).

2. **CPU requests grossly over-provisioned.** Each sbiqai pod requested `250m`
   CPU but used `1–2m` — 10 pods reserved **2500m** for ~15m of real work.
   This alone consumed ~64% of the two schedulable nodes' combined 7800m CPU
   capacity, leaving ~150m free per node — not enough for the 400m runner pod.

The third node (`ip-10-51-26-218`) had ~2700m free CPU but is fenced by
`observability=true:NoSchedule` taint (by design — dedicated to the
observability stack).

## Fix / mitigation

`kubectl patch hpa -n sbiqai-uat sbiqai-hpa --type=merge -p '{"spec":{"maxReplicas":3}}'`

Frees ~500m of CPU reservation; runner pod schedules within seconds.

## Lessons

Promoted to [[HPA memory-requests pin at max]] — the general pattern is
durable: an undersized `requests.memory` value causes memory-based HPA to
pin at max even when the workload is idle, silently consuming CPU
reservation cluster-wide.

## Follow-ups

- [ ] Raise `requests.memory` on `sbiqai` deployments (both namespaces): 256Mi → 512Mi. Expected outcome: HPA memory utilization drops from ~78% to ~40%, HPAs settle at `minReplicas: 2`.
- [ ] Lower `requests.cpu` on `sbiqai`: 250m → 50m. Real p99 usage is ~2m; 50m gives 25× headroom.
- [ ] Review whether memory-based HPA is appropriate for `sbiqai` at all — the workload is steady-state, not load-correlated. Consider CPU-only HPA.
- [ ] Add alert for pods stuck in `Pending` state > 5 minutes (this incident had zero automated detection; a developer reported it).
- [ ] Extend right-sizing review to `tasktile`, `cybersoft-dtr`, `adminsynonyms` (same pattern, smaller magnitude).

## Runbook impact

Confirms the value of [[Outage channel announcement runbook]] — the format
handled this Sev-3 case cleanly (live post + inline postmortem via `<br>`
separator).

## Related

- [[HPA memory-requests pin at max]]
- [[ccsi-msd-prd EKS cluster]]
- [[Outage channel announcement runbook]]

## Evidence (diagnostic scan, 2026-09-12)

```
Nodes (kubectl describe nodes | grep 'Taints|Allocated'):
  ip-10-51-16-160  taint=<none>                           cpu 3740m/3900m (95%)  real: 204m (5%)
  ip-10-51-26-218  taint=observability=true:NoSchedule    cpu 5280m/~8000m (66%) real: 410m (5%)
  ip-10-51-28-62   taint=<none>                           cpu 3750m/3900m (95%)  real: 132m (3%)

HPAs (kubectl get hpa -A):
  sbiqai/sbiqai-hpa       min=2 max=5 now=5   cpu 0%/70%   mem 76%/80%
  sbiqai-uat/sbiqai-hpa   min=2 max=5 now=5   cpu 0%/70%   mem 82%/80%  ← over target

sbiqai deployment resources (both namespaces):
  requests: cpu=250m memory=256Mi
  limits:   cpu=500m memory=512Mi

sbiqai pod real usage (kubectl top):
  cpu:    1–2m per pod  (~99% of reservation wasted)
  memory: 190–215Mi per pod (~78% of request → drives HPA)
```
