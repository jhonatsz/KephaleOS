---
type: incident
status: resolved
created: 2026-09-15
updated: 2026-09-15
aliases:
  - "Incident: bullzip SQS consumer stall 2026-09-15"
  - "auto_bullzip_queue_production stall"
  - "SQS bullzip task-bypass race"
tags: [sqs, bullzip, safeboxiq, taskmanager, windows-server-2016, dotnet-framework, tls, aws]
severity: SEV3
duration: unknown
sources:
  - "[[raw/notes/2026-09-14-1518-alb-listener-drift]]"
  - "[[raw/notes/2026-09-14-1548-alb-destroy-listener-inuse]]"
---

# Incident: auto_bullzip_queue_production SQS consumer stall — 2026-09-15

## Summary

The SQS queue `auto_bullzip_queue_production` in `us-west-2` accumulated 22
messages with 0 in-flight — no consumer was picking them up. Root cause was a
**business-workflow race between the bullzip "bypass task" action and the
downstream SQS consumer**: the production team bypassed a task via bullzip,
which moved the task record into a state the consumer's update-lookup step
does not handle. The consumer failed on the message, the message returned to
the queue, retries failed the same way, and the queue backed up.

Initial hypothesis was that this was fallout from the concurrent TLS 1.3 ALB
remediation on `tm.msd.cybersoftbpo.com` — but that turned out to be a
**red herring**. See [[#The TLS red herring]] below for the durable learning
that fell out of chasing the wrong cause.

## Timeline (UTC+8)

- HH:MM — Production team invoked task-bypass via bullzip on one or more
          tasks; downstream SQS message enqueued as usual
- HH:MM — Consumer picks up the first bypassed-task message, fails on the
          "look up task in bullzip and update" step
- HH:MM — Message returns to queue after VisibilityTimeout (30s); retries
          continue to fail the same way
- HH:MM — Queue depth begins climbing above baseline
- HH:MM — Depth = 22 observed via `aws sqs get-queue-attributes`
- HH:MM — Debug path 1 (WRONG): suspected fallout from TLS 1.3 remediation
          on `tm.msd.cybersoftbpo.com` because a Windows Server 2016 caller
          was also failing SSL/TLS handshake — see red herring section
- HH:MM — Root cause identified: bypassed tasks not resolvable by the SQS
          consumer's update path
- HH:MM — Applied fix: <manual reconciliation / consumer-side handling>
- HH:MM — Consumer resumed processing; queue drained to 0
- HH:MM — Incident resolved

## Impact

- 22 bullzip task updates delayed by the length of the stall
- **No message loss** — SQS retention is 4 days (default); nothing expired
- **No customer-facing outage** on `tm.msd.cybersoftbpo.com` production
- Blast radius limited to the bullzip async pipeline
- **Unrelated to** the ALB TLS 1.3 remediation on the same estate the day before

## Root cause

The bullzip "bypass task" workflow and the SQS consumer that owns the
downstream update step have **no explicit contract** for what state a task
is in after a bypass:

- Bypass moves the task record into a state the consumer's lookup treats as
  "not found" / "wrong state"
- Consumer treats not-found as a hard failure (throws) instead of a
  benign no-op (log + delete + continue)
- Message returns to queue; retries all fail the same way
- **No dead-letter queue attached** — the poison message just stalls the
  head of the queue until 4-day retention expiry
- No alarm on queue depth — issue was found by ad-hoc inspection, not by
  monitoring

## Fix / mitigation

<Fill in the exact action taken:>

- Manual reconciliation of the specific bypassed task records so consumer
  lookup succeeds, **or**
- Consumer-side code fix to treat not-found as a benign delete, **or**
- Individual message deletion via `aws sqs delete-message` after inspection

Then observed queue drain from 22 → 0 within a few minutes as consumer
picked up the remaining messages.

## Services and systems involved

**Producer side (task-bypass path):**
- **bullzip PDF printer stack** — running on Windows Server 2016 host
  `AWS-DU`. Used by internal production team to process document tasks. The
  "bypass task" action originates here.
- **taskmanager** — Rails/Node.js service behind `alb-prod-taskmanager`
  (see [[aws-tf-network]]). Business-task state authority. Bypass action
  writes to taskmanager's task record via
  `https://tm.msd.cybersoftbpo.com/`.
- **safeboxiq (sbiq-app)** — related Rails service behind
  `alb-prod-sbiq-app`. Some task workflows involve sbiq updates via
  `https://admin.safeboxiq.com/`.

**Queue:**
- **SQS `auto_bullzip_queue_production`** — us-west-2, account `872194582181`.
  Created 2021-09-22. Standard queue (not FIFO). VisibilityTimeout=30s,
  MessageRetentionPeriod=4 days. **No DLQ configured.** No CloudWatch alarm
  on depth.

**Consumer side:**
- **.NET Framework 4.7.2 service** on the Windows Server 2016 host
  `AWS-DU`. Consumes `auto_bullzip_queue_production`. Called
  `AuthenticateWorkerHandler` in application logs. **32-bit process** — this
  matters, see the red herring section.
- Consumer logic per message: read task ID from body → look up task in
  bullzip / taskmanager → apply update → delete SQS message on success.

**Network path:**
- All prod ALBs are **internal** in AWS terms, but each is fronted by an
  **internet-facing NLB** with `target_type=alb`. The Windows Server 2016
  host reaches the ALBs via public DNS through the NLB
  (see [[aws-elbv2-alb-nlb]] §4).

## The TLS red herring (durable learning)

Chasing this incident surfaced a **separate real bug** that was quietly
present on the Windows Server 2016 host:

- The **64-bit** .NET Framework registry hive
  (`HKLM:\SOFTWARE\Microsoft\.NETFramework\v4.0.30319`) had
  `SchUseStrongCrypto = 1` set.
- The **32-bit** hive (`HKLM:\SOFTWARE\WOW6432Node\...`) did **not** have it set.
- Result: 32-bit .NET Framework processes on this host default to
  `SecurityProtocolType.{Ssl3, Tls}` (TLS 1.0 + SSL 3.0 only) at ClientHello.
- Against the ALB's new `ELBSecurityPolicy-TLS13-1-2-Res-2021-06`
  policy (TLS 1.2 + 1.3 only), 32-bit .NET clients fail with
  `System.Net.WebException: Could not create SSL/TLS secure channel`.
- 64-bit .NET clients on the same host work fine — because their hive had the flag.

This was **not** the cause of the SQS backup — the queue was stalled on a
task-record state issue, not a TLS handshake. But it **would have** been
a real outage on a stricter ALB policy, and it's still a latent risk on
any other 32-bit .NET service on this host.

Durable knowledge preserved in [[dotnet-framework-tls]].

## Lessons

- **Producer-consumer contracts for workflow shortcuts (bypass, cancel,
  archive) need explicit downstream handling.** A "bypass task" action
  that leaves an already-enqueued message pointing at an invalid record is
  a well-known SQS pattern → must plan for poison messages by design, not
  by accident.
- **A poison message with no DLQ stalls the whole queue.** Head-of-line
  blocking is worse than message loss for async pipelines.
- **Client-side TLS defaults on legacy Windows can silently diverge by
  process bitness.** Registry drift between `HKLM:\SOFTWARE\Microsoft\...`
  and `HKLM:\SOFTWARE\WOW6432Node\Microsoft\...` is invisible unless you
  test both explicitly. See [[dotnet-framework-tls]] for the full pattern.
- **"Unrelated production changes" can look correlated during triage.**
  Naming the red herring explicitly in the postmortem prevents re-litigating
  the wrong cause in future retrospectives.

## Follow-ups

- [ ] Attach a dead-letter queue to `auto_bullzip_queue_production` with
      `maxReceiveCount=5`. Alarm on DLQ depth > 0.
- [ ] Add CloudWatch alarm on
      `ApproximateNumberOfMessagesVisible > 10` sustained 5 min, alert
      to on-call channel.
- [ ] Fix consumer to treat "task not in expected state / not found" as
      benign — log, delete the SQS message, continue.
- [ ] Document the bullzip "bypass task" workflow's contract with
      downstream consumers. If bypass is meant to be terminal for the
      update flow, the producing side should not enqueue the update
      message in the first place.
- [ ] Set `SchUseStrongCrypto = 1` and `SystemDefaultTlsVersions = 1`
      in `HKLM:\SOFTWARE\WOW6432Node\Microsoft\.NETFramework\v4.0.30319`
      on `AWS-DU` (and audit other Windows hosts for the same drift).
      This is defensive — closes the red-herring risk before it becomes
      a real outage on a future ALB tightening.

## Runbook impact

Suggests two new runbooks worth extracting:

- **`runbooks/sqs-poison-message-triage.md`** — how to inspect, isolate,
  and delete a poison message from a stuck SQS queue without disrupting
  in-flight processing.
- **`runbooks/windows-dotnet-tls-hardening.md`** — the SchUseStrongCrypto +
  SystemDefaultTlsVersions registry checklist for legacy Windows hosts,
  covering both 32-bit and 64-bit hives.

Not creating yet — see synthesis proposal.

## Related

- [[dotnet-framework-tls]] — durable knowledge from the red-herring debugging
- [[aws-elbv2-alb-nlb]] — ALB TLS policy behavior; the client-compat table
- [[aws-tf-network]] — repo that owns the ALB TLS policies for `tm` and `sbiq`
- Branch `devops/alb-weak-tls-remediation` on `git@git.cybersoftbpo.com:devops/aws-tf-network.git` — the 30+ commits that surrounded this incident

## Sources

- SQS queue depth observed via `aws sqs get-queue-attributes` on
  `auto_bullzip_queue_production` (us-west-2, account 872194582181), 2026-09-15.
- Windows Server 2016 host `AWS-DU`: registry inspection + PE header
  bitness check confirming the failing consumer is a 32-bit process, .NET
  Framework 4.7.2 (Release=461814).
- Test-TmTls.ps1 / Test-SbiqTls.ps1 diagnostic scripts (kept locally at
  `/Users/jhonatsz/Test-TmTls.ps1`, `/Users/jhonatsz/Test-SbiqTls.ps1` —
  reusable for future Windows-to-ALB TLS triage).
