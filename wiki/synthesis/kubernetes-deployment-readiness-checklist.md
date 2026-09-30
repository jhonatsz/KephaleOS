---
type: synthesis
status: active
created: 2026-10-01
updated: 2026-10-01
aliases:
  - "Kubernetes Deployment Readiness Checklist"
  - "K8s pre-deploy checklist"
  - "K8s readiness checklist"
tags: [kubernetes, hpa, cluster-autoscaler, capacity-planning, resource-requests, checklist]
sources:
  - "[[wiki/learnings/2026-09-12-hpa-memory-requests-pin-max]]"
  - "[[wiki/learnings/2026-09-23-requests-vs-usage-divergence-starves-cluster-capacity]]"
  - "[[wiki/learnings/2026-09-23-cluster-autoscaler-empty-nottriggerscaleup-missing-asg-tags]]"
  - "[[work/incidents/2026-09-12-gitlab-runner-cpu-reservation]]"
  - "[[wiki/technologies/ccsi-msd-prd-eks-cluster]]"
  - "[[wiki/technologies/duplo-prd-usw2-eks-cluster]]"
confidence: high
---

# Kubernetes Deployment Readiness Checklist

Pre-deploy checklist for any new or materially-modified Kubernetes
workload — the *"what should have been verified before this hit prod"*
list. Compiled from three learnings on request calibration + autoscaling
across two independent clusters, one of which produced a scheduling
outage no one saw coming.

**Use this before merging a Helm/manifest change that touches
`resources.requests`, `resources.limits`, `spec.replicas`, or any HPA.
Use it as a review lens; skip items only when you can justify why.**

---

## The failure the checklist prevents

Every item below closes one hole in a single compound failure mode:

> A deploy lands with poorly-calibrated `requests`, on a cluster whose
> elasticity mechanisms (HPA + Cluster Autoscaler) are either broken or
> blind to reservation pressure, on a cluster with no `Pending`-pod
> alerting. The scheduler exhausts its reservation budget while nodes
> idle. The next pod that tries to schedule — often an unrelated
> workload, a CI job, or a DaemonSet upgrade — fails with
> `FailedScheduling: Insufficient cpu`. Nobody pages. A developer
> notices.

This is what happened on `ccsi-msd-prd` (GitLab runner CPU reservation
incident, 2026-09-12) and what was actively happening on
`duplo-prd-usw2` (Apollo Router deploys transiently failing, 2026-09-23).
Different clusters, different teams, same shape.

---

## Pre-deploy checklist

### 1. Requests are calibrated to measured usage — not copy-pasted

- [ ] `requests.cpu` and `requests.memory` are within ~2× of P95 real
      usage (from `kubectl top` or metrics-server over ≥ 24h of
      representative load). No blind 1-CPU / 2-GiB defaults.
- [ ] If you don't have P95 telemetry for this workload yet, deploy to
      a lower environment first with a **conservative small request**
      and measure. Don't set production requests from a guess.

**Why:** Requests are a reservation contract with the scheduler, not a
comfort margin. Oversized requests exhaust reservation while nodes
idle; the HPA metric (`usage / request`) goes *down* when requests go
up, actively suppressing the rescue mechanism. See
[[Requests-vs-usage divergence starves cluster capacity]].

### 2. HPA metric matches the workload's load profile

- [ ] If the workload is **steady-state** (loaded ML models, warm
      caches, JVMs, long-lived TCP proxies), do NOT use
      memory-utilization HPA. Prefer CPU or a custom request-rate
      metric.
- [ ] If you *must* use memory-utilization HPA on a steady-state
      workload, `requests.memory` must sit comfortably above real
      steady-state memory usage — otherwise the HPA pins at max
      permanently.

**Why:** Memory-based HPA on a workload whose memory footprint doesn't
track load will silently pin at max, reserving CPU across every
replica. Failure mode is invisible until an *unrelated* pod fails to
schedule. See [[HPA memory-requests pin at max]].

### 3. Cluster Autoscaler is actually operational (not just deployed)

- [ ] If the cluster is expected to scale, verify CA can *see* the
      node groups it should manage — not just that the deployment is
      healthy.
- [ ] `NotTriggerScaleUp` events with a **blank reason** field are
      NOT a scale-up refusal. They mean CA has no idea what to scale.
      Check CA logs for `no node group config` and the AWS side for
      the tags CA's `--node-group-auto-discovery` flag expects.

**Why:** Auto-discovery is the default install pattern in every
mainstream EKS distribution — and it's silently fragile when infra
platform team owns ASGs while K8s team owns CA Helm values. Cluster
runs "healthy CA" while managing zero groups. See
[[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]].

Quick verify:

```bash
# What tags does CA expect?
kubectl -n kube-system get deploy cluster-autoscaler -o yaml \
  | grep node-group-auto-discovery

# Do any ASGs actually carry those tags?
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[?Tags[?Key==`<expected-key>` && Value==`<expected-value>`]].AutoScalingGroupName'
```

### 4. `Pending`-pod alerting exists on the target cluster

- [ ] There is an alert that fires within minutes on `phase=Pending`
      pods. Not just node CPU/memory, not just deployment
      availability — pending pods specifically.
- [ ] If not, this deploy's failure mode is *"a developer notices"*.
      That's how the 2026-09-12 CyberSoft GitLab runner incident
      surfaced — no page, no automated signal.

**Why:** Every failure in items 1–3 manifests as scheduling failures
downstream, often on *other* workloads. Without pending-pod alerting,
you learn about capacity problems through user reports, and the
misconfigured deployment is not the one that pages. See
[[Incident: GitLab Runner CPU reservation]].

### 5. Rollout headroom exists (maxSurge fits reservation budget)

- [ ] `maxSurge` + `minReplicas` × `requests.cpu` fits in remaining
      allocatable across the node pool during a rolling update.
- [ ] For a fixed-capacity cluster (no working CA), assume the cluster
      is at whatever `kubectl describe nodes` shows for
      `Allocated resources` right now — that IS your ceiling.

**Why:** Even correctly-calibrated deploys can fail transiently
mid-rollout if reservation math doesn't leave surge headroom. This is
what made every `duplo-prd-usw2` router deploy manifest
`FailedScheduling` transiently until the underlying oversized requests
were fixed. See [[duplo-prd-usw2 EKS cluster]].

### 6. Cluster context is documented and current

- [ ] Confirm target cluster has a canonical page in
      `wiki/technologies/` listing autoscaler status, alerting
      posture, known systemic issues, and namespace inventory.
- [ ] If it doesn't, the answers to items 3–5 are guesses. Write the
      page first — even a small one — so the next reviewer isn't
      re-discovering the same things.

**Why:** Both operational clusters that produced these learnings
([[ccsi-msd-prd EKS cluster]] and [[duplo-prd-usw2 EKS cluster]]) have
non-obvious systemic gaps (no autoscaler, no pending-pod alerts,
broken CA discovery, systematic CPU over-reservation) that a fresh
reviewer would not detect from the manifest alone.

---

## How to apply

**As a reviewer:** treat this as a diff-review lens. For any PR that
touches `resources.*`, `spec.replicas`, or `HorizontalPodAutoscaler`,
walk items 1–6 and leave a comment on each one that isn't clearly
addressed. Skip items only when you can name why — never silently.

**As an author:** items 1, 2, and 5 are the ones you own directly.
Items 3, 4, and 6 are cluster-posture items you probably can't fix in
this PR — but you should still know their state before shipping. If
item 3 or 4 is broken on your target cluster, the safe move may be to
delay the deploy until the cluster-side gap is closed, OR to
explicitly acknowledge that your deploy is one that tests the gap.

**As an operator:** items 3, 4, and 6 are the ones you can pre-fix
across the fleet so every downstream deploy inherits them working.
Fleet-wide `Pending`-pod alerting + CA discovery-tag validation are
the two highest-leverage fixes; both are one-time infrastructure work
that pays back on every future deploy.

---

## Counter-evidence

- **Batch/burst workloads** (CI runners, ETL jobs, scheduled ML
  training) don't fit item 2's steady-state framing — memory-based HPA
  can be appropriate for them because their memory does track load.
  The item still applies (calibrate `requests.memory` to real usage)
  but the "prefer CPU/custom metric" guidance is workload-dependent.
- **Karpenter clusters** invalidate item 3's ASG-discovery framing —
  Karpenter operates on NodePools and Provisioners, not ASG tags. The
  underlying question (*"is the elasticity mechanism actually able to
  add capacity?"*) still applies but the verify commands change.
- **Small single-tenant clusters** where reservation and utilization
  naturally track each other may not surface the item 1 failure mode.
  The checklist is still cheap to run; skipping it is only safe when
  you can articulate *why* the divergence won't grow.

---

## Evidence base

- [[HPA memory-requests pin at max]] — undersized `requests.memory`
  on memory-based HPA silently exhausts CPU reservation
- [[Requests-vs-usage divergence starves cluster capacity]] —
  oversized `requests` exhaust reservation while HPA metric stays low
- [[Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags]]
  — CA can be healthy while managing zero node groups
- [[Incident: GitLab Runner CPU reservation]] — the 2026-09-12
  incident that surfaced item 1 in production; also proved item 4's
  necessity (no pending-pod alert, developer report only)
- [[ccsi-msd-prd EKS cluster]] — cluster with no autoscaler + no
  pending alerts + systematic CPU over-reservation
- [[duplo-prd-usw2 EKS cluster]] — cluster with broken CA discovery +
  oversized Apollo Router requests

## Related

- [[aws-tf-network]] — where cluster ASG tagging typically lives at
  CyberSoft
- `Amazon EKS` — no canonical EKS technology page yet; a plausible gap
  for future capture
