---
type: learning
status: active
created: 2026-09-23
updated: 2026-10-01
aliases:
  - "Cluster autoscaler empty NotTriggerScaleUp means missing ASG discovery tags"
  - "CA silent-fail on missing ASG tags"
  - "CA empty NotTriggerScaleUp reason"
  - "cluster-autoscaler no node group config"
tags: [kubernetes, eks, cluster-autoscaler, autoscaling, asg, aws]
sources:
  - "[[duplo-prd-usw2 EKS cluster]]"
confidence: high
---

# An empty `NotTriggerScaleUp` reason means Cluster Autoscaler can't see any ASG — not that scale-up was declined

## Context

Discovered 2026-09-23 on Techstyle's [[duplo-prd-usw2 EKS cluster]]
(`duploinfra-api-us-west-2`) while investigating why the cluster ran at
97% CPU-requested and new pods sat `Pending` with `Insufficient cpu` —
despite `cluster-autoscaler` being visibly running in `kube-system`.

## What happened

CA was deployed and healthy, configured for AWS ASG auto-discovery:

```
--node-group-auto-discovery=asg:tag=duplocloud.com/eks-cluster-autoscaler/enabled=true,duplocloud.com/eks-cluster-autoscaler/cluster=$(CLUSTER_NAME)
```

But **no ASG in the account actually carried both those tags for this
cluster**. Symptoms:

- CA logs, for every node: `Node <name> should not be processed by
  cluster autoscaler (no node group config)`
- Every unschedulable pod produced a `NotTriggerScaleUp` event whose
  **reason field was blank** (literally nothing after the colon)
- Node count never changed regardless of pressure

The fix is AWS-side, not Kubernetes-side: tag the node group ASG(s)
with **both** `duplocloud.com/eks-cluster-autoscaler/enabled=true` and
`duplocloud.com/eks-cluster-autoscaler/cluster=<CLUSTER_NAME>`. Once
the tags are present, CA discovers the group on its next reconcile and
scale-up starts working — no CA restart required.

## The lesson

**An empty `NotTriggerScaleUp` reason is not a scale-up refusal — it's
a signal that CA has no idea what to scale.** Real refusal reasons are
always populated ("max node group size reached", "no matching pod
predicates", "not ready to scale up", etc.). A blank reason after the
colon means the discovery mechanism found zero matching groups, so CA
can't even *evaluate* a scale-up decision.

Reading this event as "CA looked and said no" wastes hours tuning
`skip-nodes-with-*`, `expander`, `max-node-provision-time`, or the
pod's `nodeSelector`. None of those are the problem.

## Why it matters

Auto-discovery is the default install pattern in every mainstream EKS
distribution (upstream CA Helm chart, Duplo, Karpenter's own migration
docs). Any of them can be running healthy while managing zero groups
if the ASG tags don't match — and the failure mode is silent until a
pod fails to schedule.

The pattern will recur wherever:

- CA is configured with `--node-group-auto-discovery`
- ASG tags are provisioned by a *different* team/tool than the one
  running CA (very common — infra platform team owns ASGs, K8s team
  owns CA Helm values)
- Cluster name is templated into the discovery tag but the ASG lives
  in an older/manually-provisioned account

## Detection

Two-step check:

```
# 1. What tags does CA expect?
kubectl -n kube-system get deploy cluster-autoscaler -o yaml \
  | grep node-group-auto-discovery

# 2. Do any ASGs actually carry those tags?
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[?Tags[?Key==`<expected-key>` && Value==`<expected-value>`]].AutoScalingGroupName'
```

If step 2 returns `[]`, you have this bug. Fingerprint from the cluster
side: `NotTriggerScaleUp` events with empty reason + CA logs saying
"no node group config" for every node.

## Related

- [[duplo-prd-usw2 EKS cluster]] — the cluster where this was found
- [[Requests-vs-usage divergence starves cluster capacity]] — the
  reason node pressure existed in the first place; both learnings were
  discovered in the same investigation
- [[northstar-duplo-router]] — the workload triggering scheduling failures
- [[Kubernetes Deployment Readiness Checklist]] — synthesis (item 3)
  covers verifying CA is actually operational; this learning grounds it
