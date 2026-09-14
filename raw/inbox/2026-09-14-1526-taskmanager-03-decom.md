---
type: source
status: raw
source-kind: note
created: 2026-09-14
captured-at: 2026-09-14T15:26:07
---

# Question

what the difference of alb-taskmanager and alb-taskmanager-03 and can i delete alb-taskmanager-03

# Findings

Two nearly identical stacks — but `-03` has no backend.

| Attribute | alb-prod-taskmanager | alb-prod-taskmanager-03 |
|---|---|---|
| Created | 2022-07-25 (original) | 2024-08-12 (2 years later) |
| DNS | internal-alb-prod-taskmanager-2108973080... | internal-alb-prod-taskmanager-03-198094352... |
| Target group name | tg-prod-taskmanager-rails | tg-prod-taskmanager-rails-03 |
| Registered targets | i-0358b703b60aff20c (healthy) | empty — zero targets |
| Code structure | Identical | Identical, just -03 suffix on every resource |
| Cert ARN | Same | Same |

Structurally the -03 file is a copy-paste of the original, with -03 appended to every resource name and target group name.

The -03 ALB has no backend targets — it's serving 503s to anyone who hits it. Best guesses for why it exists:
1. Unfinished blue/green cutover — someone scaffolded a second stack for a migration/canary that never completed.
2. Reserved for a second cluster — planned split of taskmanager into two pools, never actioned.
3. Just left over — templated but forgotten (created a year and a half ago, still empty).

# Can I delete alb-taskmanager-03?

YES — safe to delete. Evidence:

- Git history: Only 2 commits ever: b5e9500 (initial add, 2024-08-12) + 2ed3e9c (TLS patch) — no active work
- Cross-references in repo: None — no other TF stack references taskmanager-03
- Traffic (7 days, CloudWatch): 0 requests — completely dormant
- Registered targets: Empty — nothing to serve

But there's a pair — kill both together:

Both created the same day (2024-08-12):
- services/production/elb/alb-taskmanager-03/ (ALB)
- services/production/elb/nlb-taskmanager-03/ (NLB, internet-facing, fronting the empty ALB)

The NLB-03 is worse: it's internet-facing and pointing at an empty ALB. That's public attack surface for zero benefit.

# Recommended sequence

```bash
# 1. Destroy AWS resources
cd services/production/elb/nlb-taskmanager-03/
terraform init && terraform destroy

cd ../alb-taskmanager-03/
terraform init && terraform destroy

# 2. Once both destroys succeed, delete the TF files + S3 state
aws s3 rm s3://aws-tf-network/services/production/elb/alb-taskmanager-03/terraform.tfState
aws s3 rm s3://aws-tf-network/services/production/elb/nlb-taskmanager-03/terraform.tfState
```
