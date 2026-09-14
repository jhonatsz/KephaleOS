---
type: technology
status: active
created: 2026-09-14
updated: 2026-09-14
aliases:
  - "AWS ALB"
  - "AWS NLB"
  - "AWS ELBv2"
  - "Application Load Balancer"
  - "Network Load Balancer"
  - "ELBSecurityPolicy"
tags: [aws, load-balancer, tls, networking, security, terraform]
sources:
  - "[[raw/inbox/2026-09-14-1518-alb-listener-drift]]"
  - "[[raw/inbox/2026-09-14-1526-taskmanager-03-decom]]"
  - "[[raw/inbox/2026-09-14-1548-alb-destroy-listener-inuse]]"
  - "[[raw/inbox/2026-09-14-1550-decom-alb-tblextraction]]"
maturity: production
---

# AWS ELBv2 — ALB & NLB

## What it is

AWS Elastic Load Balancing v2 — Application Load Balancer (Layer 7,
HTTP/HTTPS) and Network Load Balancer (Layer 4, TCP/TLS/UDP). This page
captures the **non-obvious operational knowledge** — the traps you hit
in practice, not what the AWS docs describe on page one.

## Where we use it

Every CyberSoft workload sitting under `services/*/elb/` in
[[aws-tf-network]] — currently ~40+ ALBs and ~30+ NLBs across four
environments. Standard pattern is **internet-facing NLB → internal
ALB → ECS/EC2 targets**.

## Operational knowledge

### 1. Default `ssl_policy` allows TLS 1.0 / 1.1 — silently

If an `aws_lb_listener` HTTPS block omits `ssl_policy`, AWS applies
**`ELBSecurityPolicy-2016-08`** as the default. This policy still
supports **TLS 1.0 and TLS 1.1** — exactly what modern scanners flag
as *"SSL/TLS Service Supports Weak Protocol"*.

**Rule:** every `aws_lb_listener` with `protocol = "HTTPS"` must set
`ssl_policy` explicitly. Missing it is a silent security regression,
not a friendly default. First seen in [[aws-tf-network]] as 12 ALBs
across dev/uat/prod (Category A of the 2026-09 remediation).

**Preferred target policies (2026-09):**

| Policy | Protocols | Use when |
|---|---|---|
| `ELBSecurityPolicy-TLS13-1-3-2021-06` | TLS 1.3 only | Fleet-wide default when you control clients. AEAD ciphers, forward secrecy by protocol. |
| `ELBSecurityPolicy-TLS13-1-2-Res-2021-06` | TLS 1.3 preferred, TLS 1.2 restricted-cipher fallback | Safe fallback if a listener has legacy callers. Still closes weak-protocol scans. |
| `ELBSecurityPolicy-FS-1-2-2019-08` | TLS 1.2 + forward secrecy | **Not weak protocol**, but includes CBC ciphers — strict scans (Qualys/Nessus profiles) flag as weak cipher. Migrate when possible. |
| `ELBSecurityPolicy-2016-08` (default) | TLS 1.0 / 1.1 / 1.2 | **Never** — this is what "weak protocol" means. Avoid. |

### 2. TLS 1.3-only client-compat matrix

Moving to `TLS13-1-3-2021-06` cuts off any client without TLS 1.3
support. Minimum versions:

| Client | Minimum with TLS 1.3 |
|---|---|
| Chrome / Edge | 70+ (Oct 2018) |
| Firefox | 63+ |
| Safari | 14+ (macOS 11 / iOS 14) |
| Java | 8u261+, 11+ |
| .NET | .NET 4.8 partial, .NET 5+ full |
| Node.js | 12+ |
| Python | 3.7+ with OpenSSL ≥ 1.1.1 |
| curl | 7.61+ with OpenSSL ≥ 1.1.1 |
| Ruby | 2.7+ with OpenSSL ≥ 1.1.1 |
| AWS SDK / boto3 | Current versions OK; very old CI runners may not |

**Verify before applying to prod** if callers include legacy on-prem
integrations, Windows Server 2016 without patches, embedded devices,
or CI pipelines pinned to old base images. Fallback: use the `Res-2021-06`
policy instead — same scanner-clean result, keeps TLS 1.2 compat.

### 3. ALB-behind-NLB — the destroy-order trap

Pattern: an internet-facing NLB with `target_type = "alb"` in its target
group, pointing at an internal ALB. Common in the CyberSoft repos —
almost every "internal" ALB in [[aws-tf-network]] is fronted this way.

**`terraform destroy` in the ALB stack alone fails:**

```
Error: deleting Listener (...): ResourceInUse: Listener port '443' is
in use by registered target '...loadbalancer/app/<alb-name>/...' and
cannot be removed.
```

The confusing phrasing — "in use by registered target" — means **the
ALB is registered as a target of the NLB's target group**. AWS
prevents the ALB from being removed while another load balancer points
at it.

**Correct destroy order — NLB first, ALB second:**

```bash
cd services/<env>/elb/nlb-<service>/
terraform init && terraform destroy   # removes NLB + ALB-as-target registration

cd ../alb-<service>/
terraform destroy                      # now succeeds
```

Applies to every ALB decom in [[aws-tf-network]]. Confirmed hits: 2026-09-14
on `alb-prod-unstractlte` and `alb-tblextraction`.

### 4. "Internal" ALB fronted by "internet-facing" NLB — scan-reachability paradox

An ALB with `Scheme = internal` in AWS looks safe on paper — no public
DNS, no public IP. But if it's the target of an **internet-facing** NLB,
scanners on the public internet reach the ALB's TLS termination via the
NLB's public DNS. The NLB is a TCP passthrough — it does not terminate
TLS. TLS handshake happens on the ALB.

**Consequence:** weak-protocol findings on "internal" ALBs are real
public exposure, not internal-scanner artefacts. When triaging a scan
result, check for an NLB twin:

```bash
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?Scheme==`internet-facing`].[LoadBalancerName,DNSName]' \
  --output table
```

Cross-reference with target groups where `TargetType = alb`.

### 5. Persistent listener-`default_action` representation drift

Every `terraform plan` on an ALB listener where the code uses a
`forward { target_group { arn = ... } }` block will show:

```
~ default_action {
    - target_group_arn = "arn:...:targetgroup/..." -> null
    + forward {
        + target_group {
            + arn    = "arn:...:targetgroup/..."
            + weight = 1
          }
      }
  }
```

This is **not real drift.** State stores the older `target_group_arn`
shorthand; TF code uses the newer nested `forward` block. Applying just
switches the state representation — same target group, same routing,
zero traffic impact. Ignore in plan output. Present in ~every ALB in
[[aws-tf-network]].

### 6. Health-check settings are frequently tuned in the console

Multiple stacks in [[aws-tf-network]] have `health_check.interval` and
`health_check.timeout` values in AWS that differ from TF (e.g. prod
sbiq-app: TF said 5s/3s, AWS ran at 60s/10s). These are usually
**intentional** tuning by operators to reduce probe noise. Adopt into
TF rather than "fix" back to TF values — reverting can cause spurious
target unhealthy flips under load.

Same rule applies to `idle_timeout` on the ALB itself (seen at 300,
600, 900 across the fleet).

## Common issues → solutions

- **Scanner reports "SSL/TLS Service Supports Weak Protocol" on an ALB**
  → set `ssl_policy` explicitly on the listener. If it was empty, AWS
  was applying `ELBSecurityPolicy-2016-08`. Target `TLS13-1-3-2021-06`
  or `TLS13-1-2-Res-2021-06`.
- **`terraform destroy` of ALB fails with `ResourceInUse`** → NLB in
  front. Destroy NLB stack first.
- **Every plan shows `default_action` diff** → representation drift,
  not real. Apply is harmless.
- **Every plan shows manual-tag drift** → someone tagged the resource
  in the console. Adopt into TF; don't fight it.

## Related

- [[aws-tf-network]] — primary consumer of this pattern in CyberSoft.
- PCI DSS 4.0 §4.2 — mandates TLS 1.2 minimum for cardholder data
  transmission over open networks. `ELBSecurityPolicy-2016-08` fails
  this control.

## Sources

- 2026-09-13 → 2026-09-14 CyberSoft `devops/alb-weak-tls-remediation`
  branch — 26 commits, patched 19 live ALBs across dev/staging/uat/prod.
- Inbox captures from the same session:
  - [[raw/inbox/2026-09-14-1518-alb-listener-drift]]
  - [[raw/inbox/2026-09-14-1526-taskmanager-03-decom]]
  - [[raw/inbox/2026-09-14-1548-alb-destroy-listener-inuse]]
  - [[raw/inbox/2026-09-14-1550-decom-alb-tblextraction]]
- AWS docs: [ELB security policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html#describe-ssl-policies)
