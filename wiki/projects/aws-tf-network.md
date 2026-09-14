---
type: project
status: active
created: 2026-09-13
updated: 2026-09-14
aliases:
  - "AWS TF Network"
  - "aws-tf-network"
  - "atn"
  - "Cybersoft AWS Network"
  - "vpc01"
tags: [terraform, aws, vpc, networking, cybersoft, hybrid-connectivity, iac]
repo: git@git.cybersoftbpo.com:devops/aws-tf-network.git
time-sensitive: true
confidence: high
---

# Project: aws-tf-network

## Purpose

Terraform-managed AWS network for CyberSoft: a **single VPC**
(`10.51.0.0/16`, `us-west-2`) that hosts four logical environments —
`develop`, `staging`, `uat`, `production` — as subnet tiers, plus
site-to-site VPN to on-prem and peering to four external AWS VPCs.
All ~97 workload load balancers in the `services/` tree depend on
this stack.

## Context

- Organization: CyberSoft (DevOps)
- Repository: `git@git.cybersoftbpo.com:devops/aws-tf-network.git`
- Local working dir: `/Users/jhonatsz/workspace/cybersoft/aws-tf-network`
- AWS profile: `cybersoftbpo-jhf` · region: `us-west-2`
- State backend: S3 bucket `aws-tf-network`, DynamoDB lock table
  `atn-terraformLock`, SSE-S3
- Terraform: `>= 0.14.9` · AWS provider: `~> 3.27` (both aging —
  worth flagging on any upgrade project)

## Architecture (durable knowledge)

**Single-VPC, multi-env-by-subnet-tier.** This is the load-bearing
design choice. Environments are **not** separate VPCs; they share
`10.51.0.0/16` and are separated by subnet tier + route table. Each
non-public env has three tiers: `apps` (ECS / app VMs), `rds`, `redis`.
Trade-off: cheaper and simpler, but `develop` and `staging` end up
sharing CIDR space, and any cross-env blast-radius is a single VPC
away.

**Public tier is shared** across environments — two `/24`s
(`10.51.100.0/24` in `us-west-2a`, `10.51.101.0/24` in `us-west-2b`),
carrying ALBs/NLBs and the two NAT gateways. One NAT per AZ; each has
its own EIP.

**Env subnet ranges (as of 2026-09-13):**

| Env             | rds `/24`s   | apps `/24`s     | redis `/24`s   |
|-----------------|--------------|-----------------|----------------|
| develop+staging | 106, 107     | 168, 169        | 231, 232       |
| uat             | 76, 77       | 131, 132        | 201, 202       |
| production      | 86, 87       | 148, 149        | 210, 211       |

All in `10.51.<x>.0/24`, split across `us-west-2a`/`2b`.

**Hybrid connectivity.**
- One VPN Gateway attached to the VPC (`vgw-*` id hardcoded in
  `networks/vpc01/vpc/main.tf`).
- Two active Customer Gateways: `atn-wa-rise` and `atn-wa-pldt`
  (both BGP ASN 65001). Two more (`atn-ec-telstra`, `atn-ec-globe`)
  exist as commented files.
- On-prem CIDRs routed via VGW: `10.0.0.0/16`, `10.1.0.0/16`,
  `192.168.0.0/21`. Wired for UAT and PROD private tiers; **dev has
  the VPN routes commented out** — intentional-or-drift question,
  not yet answered.

**VPC peerings** (four; peering resources not managed here — this
repo only routes to them):

| Remote CIDR    | Used by                                       |
|----------------|-----------------------------------------------|
| `10.47.0.0/16` | Public RT, PROD apps                          |
| `10.48.0.0/16` | Public RT, UAT apps/rds/redis, PROD apps/rds  |
| `10.52.0.0/22` | Public RT                                     |
| `10.79.0.0/16` | Public RT                                     |

Peering IDs (`pcx-*`) are hardcoded literals in multiple files.

**Repo layout.** Each leaf directory containing `backend.tf` is an
independent stack — `terraform init/plan/apply` runs inside it
directly. Stacks: `networks/vpc01/{vpc,subnets,security-groups,
nat-gateways,eips,vpn-gateways,customer-gateways}`, then
`services/<env>/elb/<lb>/` per load balancer.

**Services tree layout.** `services/<env>/ecs/` splits into subdirs
by resource type: `alb/`, `clusters/`, `services/` (per-app ECS
service dirs), and a legacy `service/` (**singular**) tree that
predates the plural convention. **New ECS services go in `services/`
(plural).** The legacy `service/` singular dirs were being retired
during 2026-09 — same env can have both trees during the transition,
which is exactly how the within-env `aws_ecs_service.name` collisions
(see DO-1922) arose. RDS/redis sit under
`services/<env>/{rds,redis}/<db-or-cache>/`.

**Apply order (from-scratch, rarely needed):** vpc → eips → cgws →
vgws → nat-gateways → subnets → security-groups → per-LB services.

**Not in this repo.** Route 53 private zones, VPC endpoints (S3,
DynamoDB, SSM, ECR — placeholders exist but no resources), Direct
Connect / Transit Gateway, IAM roles, KMS keys. If those exist,
they live elsewhere.

## Conventions

- **Branch:** `devops/DO-<ticket>-<short-description>`
- **Commit subject:** `ops: DO-<ticket> <short description>`
- **Tags on resources:** `Name = atn-<Kind>-<qualifier>`,
  `Environment`, `Service`, `Maintainer = Terraform` (last one added
  by every module automatically).
- **ECS service `name` uniqueness.** `aws_ecs_service.name` must be
  unique per env (per AWS account). A collision inside one env silently
  breaks `terraform apply` on the second stack. Sanity check for a
  given env with:
  ```
  grep -rn 'name.*=.*"[^"]*service"' services/<env>/ecs/
  ```
  Cross-env repetition (same name in `develop` / `uat` / `production`)
  is intentional — different backend states, different AWS accounts.
- **Safe cleanup workflow for legacy stacks.** Operator runs
  `terraform destroy` per stack; only after state is torn down does
  the `.tf` config get removed. One commit per env-scope so any
  rollback is surgical. Deleting `.tf` before `destroy` orphans state
  and leaves live infra unmanaged.
- **ALB-behind-NLB destroy order.** Destroy the NLB stack **before**
  the ALB stack. See [[aws-elbv2-alb-nlb]] §3 for the full trap.
- **Manual tag policy is applied via AWS console after TF creation.**
  Every ALB/target-group plan shows drift for `Department` / `Team` /
  `Owner` / `ManagedBy` plus lowercase `Environment` / `Project`.
  **Do not "fix" this back to the TF values** — adopt the manual tags
  into TF so plans go clean. Team-ownership map, as observed 2026-09-14:

  | Service family | `Department` | `Team` |
  |---|---|---|
  | taskmanager · encodingtool · esfuse · sbiq-app · staging-blobman | `mis` | `mis-msd` |
  | nationwideloans · blobman (dev/prod) · conditionsai | `de-ai` | `ai` |
  | airflow-llmsplitter (uat/prod) | `de-ai` | `de` |
  | unstract-lte | `sharedresource` | *(no Team tag)* |

  Standard reconciled tag block:
  ```hcl
  tags = {
    Project     = "<lowercase-with-hyphens>"
    Environment = "<lowercase-env>"
    Department  = "<from table>"
    Team        = "<from table>"    # omit if sharedresource
    Owner       = "DevOps"
    ManagedBy   = "Terraform"
  }
  ```

  Only `aws_lb` and `aws_lb_target_group` tags show drift in practice;
  listener tags in AWS still match the pre-drift TF values, so leave
  them alone.
- **Verify `aws_lb` liveness before `terraform apply` on any stack.**
  Many `services/*/elb/` stacks are ghost TF — code exists but the
  ALB was never created (or was destroyed without cleanup). Running
  `terraform apply` in a ghost stack creates a **new** ALB you don't
  want. Verify first:

  ```bash
  aws elbv2 describe-load-balancers --names <alb-name> \
    --region us-west-2 --profile cybersoftbpo-jhf \
    --query 'LoadBalancers[0].State.Code' --output text
  ```

  Observed 2026-09-14: 6 of 12 Category A stacks + 13 of 28 Category B
  stacks were ghost — TF exists, no live AWS resource. Candidates for
  cleanup rather than apply.

## Key decisions

*(No ADR-style decisions captured yet in `wiki/decisions/`. The
"single VPC with subnet-tier envs" choice deserves a retroactive ADR
if this project ever gets touched heavily.)*

## History (durable to remember)

- **2026-09-13 → 2026-09-14 · SSL/TLS weak-protocol remediation.**
  Branch `devops/alb-weak-tls-remediation` (26 commits). Fleet-wide
  bump of all live ALB listeners to `ELBSecurityPolicy-TLS13-1-3-2021-06`
  (TLS 1.3 only). Two problem classes fixed:

  **Category A (12 files, 6 live ALBs):** HTTPS listener had no
  `ssl_policy` at all — AWS was silently applying
  `ELBSecurityPolicy-2016-08` (allows TLS 1.0/1.1). This was the
  root cause of the scanner finding. Details in [[aws-elbv2-alb-nlb]] §1.

  **Category B (28 files, 15 live ALBs):** Previously on
  `ELBSecurityPolicy-FS-1-2-2019-08` (TLS 1.2 + FS, includes CBC ciphers).
  Not weak protocol, but bumped for consistency and to survive strict
  cipher scans.

  **Applied to 19 ALBs; 2 decommissioned mid-sweep** (`alb-prod-unstractlte`,
  `alb-tblextraction`). One ghost-stack cleanup done in-branch
  (`services/develop/elb/alb-sbiq-app/` removed).

  **Bundled side-fixes** during the sweep:
  - Reconciled manual tag drift on 15 stacks (see Conventions).
  - Fixed misspelled tag `Deparment` → `Department` on `alb-prod-nwl-app`.
  - Preserved health_check tuning on prod sbiq-app (60s/10s vs TF's 5s/3s).
  - Preserved `idle_timeout = 600` on staging sbiq-app.

  **Follow-ups queued** — decom candidates for a separate PR:
  `alb-prod-taskmanager-03` (empty target groups, 0 traffic since
  2024-08 creation) plus its `nlb-prod-taskmanager-03` twin.

- **2026-09-13 · DO-1922 · Duplicate-service cleanup + truoffer
  decommission.** Retired the legacy `services/<env>/ecs/service/`
  (singular) tree in develop / uat / production by moving workloads
  into the plural `services/` tree, resolving two within-env
  `aws_ecs_service.name` collisions (prod `ec-rails-service` between
  metabase-pro and encoding-tool → kept encoding-tool; uat
  `airflow-webhub-service` between standalone airflow-webhub and
  sbiqlite-airflow → kept sbiqlite-airflow), and fully decommissioned
  the **truoffer** project in both prod and uat (ECS service, ECS
  cluster, RDS, ALB, KMS). Metabase Pro in prod was also
  decommissioned in the same pass. Branch:
  `devops/DO-1922-cleanup-duplicate-services-and-truoffer`.

## Known issues (compiled 2026-09-13)

Surfaced while writing `docs/known-issues.md` in the repo. All are
"non-obvious, will bite a new engineer" — durable to remember.

1. **AZ region mismatch.**
   `networks/vpc01/subnets/subnets-public-all.tf:18` uses
   `length(["us-west-1a", "us-west-1b"])` for the public-RT
   association count. The value (`2`) is correct, but the literal is
   `us-west-1` while the whole VPC is `us-west-2`. Cosmetic today,
   copy-paste hazard tomorrow.
2. **VPC endpoints declared but empty.**
   All private-app route tables carry `vpc_endpoint_id = ""`
   placeholders. No `aws_vpc_endpoint` resources exist. Every S3/ECR/
   SSM/DynamoDB call exits through NAT — real NAT cost that a gateway
   endpoint for S3 would eliminate for ~free.
3. **Peering IDs hardcoded** in `networks/vpc01/vpc/main.tf` *and*
   duplicated in `subnets-private-uat-apps.tf`, `subnets-private-prod-apps.tf`.
   Any peering recreation is an N-place edit.
4. **Dev VPN routes commented out** in `subnets-private-dev-apps.tf:36-48`.
   UAT and PROD have them. Unclear if `develop` is deliberately
   isolated from on-prem or if this is drift.
5. **Develop and Staging share subnet CIDRs.** Only one can be
   active at a time. There is no separate staging subnet stack.
6. **Root README claims subnets that don't exist in code.**
   `atn-psn-2a` (`10.51.103.0/24`) and `atn-psn-2b` (`10.51.104.0/24`)
   are listed under GLOBAL but no Terraform creates them. Corrected
   in the rewritten README.
7. **`.gitlab-ci.yml` is a stub** (`echo "Hello World"`). No
   `terraform fmt/validate/plan` on MRs.
8. **No `.tflint.hcl` / `.terraform-docs.yml` / `.pre-commit-config.yaml`.**
   No lint, no auto-doc.
9. **Ghost TF stacks under `services/*/elb/`.** ~19 of 40+ stacks have
   TF but no live AWS ALB. `terraform apply` in a ghost stack creates
   a rogue LB. Discovered while running the 2026-09 TLS remediation —
   always `aws elbv2 describe-load-balancers` before applying. See
   Conventions section for the check.
10. **`alb-prod-taskmanager-03` zombie.** Structural clone of
    `alb-prod-taskmanager` (2024-08-12) with 0 targets and 0 traffic
    since creation. Its NLB twin `nlb-prod-taskmanager-03` is
    internet-facing and pointing at nothing — public attack surface
    for zero benefit. Decom candidate.

## Documentation state (post-2026-09-13 pass)

The repo's docs were rewritten this session. Durable to remember:

- `README.md` — rewritten as an index (was mostly a CIDR table with
  gaps).
- `docs/architecture/network.md` — narrative + two embedded Mermaid
  diagrams.
- `docs/architecture/diagram-topology.mmd` and `diagram-routing.mmd`
  — source of truth for the diagrams.
- `docs/architecture/cidr-allocation.md` — authoritative CIDR table
  + free-`/24` list + add-a-subnet checklist.
- `docs/onboarding.md` — AWS profile, backend, stack→state-key map,
  workflow, commit conventions.
- `docs/known-issues.md` — the eight items above, in the repo.

Not done (deliberate follow-ups): per-module READMEs, terraform-docs,
pre-commit hooks, real `.gitlab-ci.yml`.

## Related

- [[aws-elbv2-alb-nlb]] — AWS ALB/NLB operational patterns (weak-default
  `ssl_policy`, ALB-behind-NLB destroy trap, TLS 1.3 client compat,
  listener representation drift) — the durable technical knowledge
  behind the 2026-09 remediation.
- [[k8s-gitlab-runner]] — sibling CyberSoft DevOps project; workloads
  in `services/` may be deployed by this runner.
- [[Runbook: GitLab Runner token rotation]] — same GitLab instance
  (`git.cybersoftbpo.com`).

## Sources

- Local repo: `/Users/jhonatsz/workspace/cybersoft/aws-tf-network`
- Git remote: `git@git.cybersoftbpo.com:devops/aws-tf-network.git`,
  branch `master` as of 2026-09-13 (clean, last commit `19b31df`).
- Compiled from a full repo scan on 2026-09-13: every `.tf` in
  `networks/vpc01/`, the `modules/` tree, all four `services/*/`
  trees, and the last ~30 commits.
- Repo-local docs authored in the same session:
  `docs/architecture/network.md`, `docs/architecture/cidr-allocation.md`,
  `docs/onboarding.md`, `docs/known-issues.md` — these are the
  ground-truth references, not this page.
