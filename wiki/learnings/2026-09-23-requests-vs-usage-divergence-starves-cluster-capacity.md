---
type: learning
status: active
created: 2026-09-23
updated: 2026-10-01
aliases:
  - "Requests-vs-usage divergence starves cluster capacity"
  - "Oversized CPU requests starve scheduler"
  - "Reservation vs utilization divergence"
tags: [kubernetes, capacity-planning, resource-requests, hpa, scheduler]
sources:
  - "[[northstar-duplo-router]]"
  - "[[duplo-prd-usw2 EKS cluster]]"
confidence: high
---

# When container `requests` sit far above real usage, the scheduler's reservation budget runs out while nodes idle — and HPA can't compensate

## Context

Discovered 2026-09-23 while investigating why the Apollo Router tenant
`gql-usw2` on Techstyle's [[duplo-prd-usw2 EKS cluster]] couldn't
complete a rolling deploy without pods sitting `Pending`. Nodes were
visibly idle in `kubectl top`, yet the scheduler reported no room.

## What happened

Router pods (see [[northstar-duplo-router]]) requested **1 CPU and
2 GiB memory** each; measured usage sat at **~50m CPU and a few hundred
MiB per pod**. Concretely:

- HPA min-replicas: **6** → reserves 6 CPU on a cluster with ~11.7
  allocatable CPU across three nodes (~51% of total capacity, before
  any other workload)
- Rolling deploy adds `maxSurge` pods, briefly pushing reservation
  over what the cluster can hold
- Result: cluster ran at **~97% CPU-*requested*** while nodes were
  **7–11% actually utilized**
- Every deploy manifested transient `FailedScheduling: Insufficient cpu`
  events despite the workload doing almost nothing

**HPA never scaled up.** The metric is `usage / request`. With 50m real
against 1000m request, the metric hovered near 5% — nowhere near the
60%-CPU / 75%-memory targets. So the HPA that would normally add
capacity under load was blind to the actual pressure, because the
pressure was on *scheduler reservation*, not on the pods themselves.

Node capacity couldn't grow either — the cluster autoscaler was
misconfigured (see [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]]).
So the same investigation surfaced both a bad request calibration and
a broken elasticity mechanism, stacked.

## The lesson

**Container `requests` are a reservation contract with the scheduler,
not a comfort margin.** When requests are set well above real usage:

1. The scheduler consumes reservation budget for capacity that will
   never be used.
2. Pods fail to schedule while nodes visibly idle in `top`.
3. HPA cannot rescue the situation — its metric goes DOWN when
   requests go UP (`usage / request`), so oversized requests actively
   suppress the autoscaler that would otherwise add pods.
4. Cluster Autoscaler cannot rescue the situation either — it acts on
   *pending pods*, not on reservation percentage; and if a rolling
   deploy is the trigger, CA's scale-up latency is often longer than
   the deploy timeout.

Right-size requests to **P95 of real usage plus a small headroom**, not
to peak-worst-case-forever. If you don't know P95, the deploy is not
production-ready.

## Why it matters

This is the *opposite* failure mode of
[[HPA memory-requests pin at max]] (requests too *low* → HPA fires
always). Both are calibration failures on the same knob. Together they
form a diagnostic frame: **before touching HPA thresholds or node
counts, verify `requests` are within striking distance of measured
usage.** Everything else is downstream of that.

The pattern will recur wherever:

- Teams inherit deploys with legacy request values that were sized for
  a heavier historical workload
- "Safe" defaults get copy-pasted across services regardless of profile
- Requests were set once, at deploy time, and never revisited against
  actual telemetry

## Detection

```
# Compare requested vs used at the pod level
kubectl top pods -A --containers \
  --sort-by=cpu | head -20

# Compare requested vs allocatable at the node level
kubectl describe nodes | grep -E "(Name:|cpu\s+[0-9]+)"

# Or via metrics-server + a script:
# For each pod, (requests.cpu, usage.cpu, ratio). Any ratio > 5
# is a candidate for right-sizing.
```

Fingerprint: cluster shows high `CPURequested`% in
`kubectl describe nodes` while `kubectl top nodes` shows low actual %.
The divergence — not either number alone — is the signal.

## Related

- [[HPA memory-requests pin at max]] — the opposite calibration failure
  on the same knob (learning pair)
- [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]] —
  discovered in the same 2026-09-23 investigation
- [[northstar-duplo-router]] — workload where this was first seen
- [[duplo-prd-usw2 EKS cluster]] — cluster context
- [[Kubernetes Deployment Readiness Checklist]] — synthesis (item 1)
  covers request calibration; this learning grounds it
