---
type: source
status: raw
source-kind: note
created: 2026-09-14
captured-at: 2026-09-14T15:50:09
---

# decommission alb-tblextraction

alb-tblextraction

## Destroy order (ALB-behind-NLB pattern in aws-tf-network repo)

```bash
cd services/production/elb/nlb-tblextraction/
terraform init && terraform destroy   # kill NLB first

cd ../alb-tblextraction/
terraform destroy                      # then ALB
```

If you destroy the ALB first: fails with `ResourceInUse: Listener port '443' is in use by registered target` because `nlb-tblextraction` still has this ALB registered as a target (target_type=alb).

Same pattern applied to alb-prod-unstractlte earlier.
