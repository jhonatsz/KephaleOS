---
type: runbook
status: active
created: 2026-09-13
updated: 2026-09-13
aliases:
  - "Ops communication templates"
  - "Maintenance announcement template"
  - "Change management template"
  - "Incident / outage template"
  - "Postmortem template"
tags: [ops, incident-communication, change-management, maintenance, postmortem, portable]
---

# Runbook: Generic ops communication templates

Reusable operational communication templates for any project or client
engagement. Three connected artifacts covering the full lifecycle:

1. **Maintenance announcement** — planned work notice in a maintenance channel
2. **Change management** — CHG record, pre-flight, post-change review
3. **Outage / incident** — declaration, comms, resolution, postmortem

**Sibling — internal-flavored variant:** for the specific cybersoft
`! IMP - Outages` channel format (STATUS-tagged, HTML `<br>` combined
postmortem), use [[wiki/runbooks/outage-channel-announcement]] instead.
This page is the portable trio for other contexts.

## When to use

- Setting up ops process at a new client / new project.
- Standardizing comms after a first messy incident revealed the gap.
- Answering the recurring question *"what should the maintenance / change /
  outage post look like?"*

## Design principles (durable — the "why")

- **Rollback is a first-class deliverable.** No rollback plan → not approved,
  not shipped. This applies to change requests AND maintenance announcements.
- **Decision criteria written before the change.** Prevents "let's try one
  more thing" mid-outage. The trigger for rollback is a number and a
  duration, not vibes.
- **"No update" is an update.** For live incidents, silence panics
  stakeholders. Every ongoing update carries the next-update timestamp.
- **Owner named, always.** Ambient ownership = no ownership.
- **IC ≠ operator.** For SEV1/2, the incident commander coordinates and
  does not type commands. If IC starts debugging, IC has lost the plot —
  hand off. This is enforced structurally, not culturally.
- **Blameless postmortems.** Language is about systems and process, not
  people ("the deploy pipeline allowed…" not "Alice pushed…").
- **Timeline before analysis.** Facts first, story after — prevents
  narrative bias in the postmortem.
- **"Where we got lucky"** — surfaces near-miss risk that plain root-cause
  hides. It's where the real investment case lives.
- **Every action item has an owner, priority, and due date.** No orphans.
- **Security lens is mandatory, not optional.** Every incident asks "is
  this a breach?" — default answer isn't "no," it's "prove it."

---

## Template 1 — Maintenance channel announcement

### Planned maintenance

```
🛠️ [SCHEDULED] <service> — <short title>

When:      <YYYY-MM-DD HH:MM–HH:MM TZ>  (<duration>)
Impact:    <who/what is affected — be concrete>
Downtime:  <yes / no / partial — e.g. "5-min API blip at start">
Reason:    <one line — why this is happening>
Owner:     @<person>  |  Backup: @<person>
Runbook:   <link>
Status:    <link to status page / incident channel>

Rollback:  <how we back out if it goes wrong, ETA>
```

### Emergency / unplanned

```
🚨 [ONGOING] <service> — <one-line symptom>

Started:   <HH:MM TZ>
Impact:    <user-facing effect>
Status:    Investigating | Identified | Mitigating | Monitoring | Resolved
Owner:     @<person>
Updates:   every <15/30> min in this thread
Next update: <HH:MM TZ>
```

### Update reply (in-thread)

```
[HH:MM TZ] <Status>: <what changed since last update>
Next update: <HH:MM TZ>
```

### Resolved

```
✅ [RESOLVED] <service> — <title>
Duration:  <HH:MM–HH:MM TZ> (<X min>)
Impact:    <final scope>
Cause:     <one line>
Follow-up: <postmortem link / due date>
```

---

## Template 2 — Change management

### Change request

```
Change ID:   CHG-<YYYY-MMDD-###>
Title:       <verb + system + outcome>
Type:        Standard | Normal | Emergency
Risk:        Low | Medium | High
Priority:    Low | Medium | High

── Context ──
What:        <one paragraph — what's being changed>
Why:         <business/tech driver — link ticket, incident, or ADR>
Systems:     <services / envs / regions affected>
Users:       <who's impacted, internal & external>

── Plan ──
Window:      <YYYY-MM-DD HH:MM–HH:MM TZ>
Steps:       1. <step> — <who> — <duration>
             2. …
Validation:  <how we know it worked — checks, dashboards, smoke tests>

── Risk ──
Blast radius:   <worst case if this goes sideways>
Rollback plan:  <exact steps, ETA, decision criteria>
Rollback owner: @<person>
Dependencies:   <upstream/downstream systems, freezes, other CHGs>

── People ──
Implementer:  @<person>
Approver:     @<person>  (required for Normal/High-risk)
Reviewer:     @<peer>
Comms:        @<person>

── Comms plan ──
T-72h: notice to <audience>
T-24h: reminder in maintenance channel
T-0:   start announcement
T+X:   completion / rollback announcement
```

### Pre-flight checklist

```
[ ] Approver signed off
[ ] Rollback tested (or documented why not testable)
[ ] Backups verified for anything mutable
[ ] Monitoring/alerts confirmed working on target systems
[ ] Announcement posted in maintenance channel
[ ] Runbook open, implementer + reviewer both present
[ ] No conflicting change in same window
[ ] Freeze windows checked (release freeze, holiday, on-call handoff)
[ ] Decision criteria for rollback are written down, not vibes
```

### Post-change review

```
Change ID:      CHG-<...>
Outcome:        Success | Partial | Rolled back | Failed
Actual window:  <HH:MM–HH:MM TZ>  (planned: <HH:MM–HH:MM>)
Deviations:     <what didn't go per plan>
Incidents:      <any INC-# opened during/after>

Lessons:
- <what worked>
- <what surprised us>
- <what we'd do differently>

Follow-ups:
- [ ] <action> — @<owner> — <due>
```

### Change type — decision rule

| Type | When | Approval |
|---|---|---|
| **Standard** | Pre-approved, repeatable, low-risk (e.g. cert renewal via known runbook) | Auto — implementer's call |
| **Normal** | Everything else — new work, config changes, deploys with real blast radius | Approver + reviewer |
| **Emergency** | Prod is broken or breach in progress; change is the mitigation | Post-hoc review within 24h |

---

## Template 3 — Outage / incident

### Incident declaration

```
Incident ID:  INC-<YYYY-MMDD-###>
Title:        <symptom + system>
Severity:     SEV1 | SEV2 | SEV3 | SEV4
Started:      <YYYY-MM-DD HH:MM TZ>  (detected: <HH:MM>)
Detected by:  <alert name / user report / dashboard>
Status:       Investigating | Identified | Mitigating | Monitoring | Resolved

── Impact ──
Users:        <who / how many / which segment>
Services:     <affected systems, regions>
Business:     <revenue / SLA / compliance clock ticking?>
Security:     Yes | No | Unknown  ← if Yes or Unknown → page security lead

── Channels ──
War room:     #inc-<id>
Bridge:       <video link if SEV1/2>
Status page:  <link if customer-facing>
Related CHG:  <CHG-id if this traces to a change>
```

### Severity — decision rule

| SEV | Trigger | Response |
|---|---|---|
| **SEV1** | Major outage, revenue-impacting, security breach, data loss | Page immediately, all-hands, bridge open, exec notified |
| **SEV2** | Significant degradation, some users blocked, workaround exists | Page on-call, war room in Slack, exec informed within 1h |
| **SEV3** | Minor degradation, few users affected, non-urgent | Ticket, business-hours response |
| **SEV4** | Cosmetic / no user impact | Backlog |

Escalate to SEV1 immediately if: security breach suspected, data exfiltration
possible, regulatory clock started, or you're not sure. Err up, not down.

### Roles (SEV1/SEV2)

```
Incident Commander (IC):  @<person> — runs the response, makes calls
Comms Lead:               @<person> — updates, status page, stakeholders
Ops Lead:                 @<person> — hands on keyboard, mitigations
Scribe:                   @<person> — timeline, decisions, actions
Security Lead:            @<person> — if breach-adjacent
Exec liaison:             @<person> — SEV1 only, shields IC from exec pings
```

### Update cadence

```
SEV1: every 15 min, even if "no change"
SEV2: every 30 min
SEV3: every 2h or on status change
```

Update format:

```
[HH:MM TZ] <Status>: <what we know / what we tried / what's next>
Impact now: <current scope>
Next update: <HH:MM TZ>
```

### External comms templates

**Initial:**

```
We are investigating reports of <symptom> affecting <scope>.
Started at <HH:MM TZ>. We'll post the next update by <HH:MM TZ>.
```

**Identified:**

```
We've identified the cause of <symptom> and are working on a fix.
Impact: <scope>. Next update by <HH:MM TZ>.
```

**Resolved:**

```
The issue affecting <scope> was resolved at <HH:MM TZ>.
Duration: <X min>. A postmortem will be published within <5 business days>.
```

**Internal SEV1 exec ping:**

```
SEV1 declared. INC-<id>. <one-line symptom>. IC: @<name>. War room: #inc-<id>.
Customer-facing: yes/no. Security-adjacent: yes/no. First update by <HH:MM>.
```

### Resolution

```
Resolved at:    <HH:MM TZ>
Total duration: <HH:MM>  (detect → resolve)
TTR breakdown:  Detect: <X min>  |  Diagnose: <Y min>  |  Fix: <Z min>
Root cause:     <one line — code / config / infra / process / human>
Mitigation:     <what we did to stop the bleeding>
Fix:            <permanent fix, if different from mitigation>
Verification:   <how we confirmed resolution>

Follow-ups filed:
- [ ] <action> — @<owner> — <due> — <ticket>
```

### Postmortem (blameless — due within 5 business days)

```
── Header ──
Incident:    INC-<id> — <title>
Severity:    SEV<n>
Date:        <YYYY-MM-DD>
Duration:    <HH:MM>
Author:      @<person>
Reviewers:   @<people>

── Summary ──
<3-4 sentences: what happened, who was affected, what we did, when it ended>

── Impact ──
- Users affected: <number / segment>
- Duration of user impact: <X min>
- Business impact: <revenue / SLA / trust>
- Data loss: yes / no — <details>
- Security impact: yes / no — <details>

── Timeline (TZ) ──
HH:MM — <event> — <who noticed / did what>

── Root cause ──
<Technical explanation. Include the "5 whys" if it clarifies.>

── What went well ──
- <specific things>

── What went poorly ──
- <specific gaps>

── Where we got lucky ──
- <what could have been much worse — worth naming, drives investment>

── Action items ──
| # | Action | Owner | Priority | Due | Ticket |
|---|--------|-------|----------|-----|--------|
| 1 | <fix root cause> | @… | P0 | <date> | <link> |

── Lessons ──
<What we now understand about the system that we didn't before.>
```

---

## Worked samples — connected narrative

The three samples below share one story so you can see how the templates
cross-reference in practice. Scenario: **Postgres 14→15 major upgrade on prod
primary that fails on cutover** because pgbouncer's `userlist.txt` still had
md5 password hashes while PG15 defaulted to `scram-sha-256`. 47-minute
customer-facing incident, ~€38k delayed revenue, brushed the enterprise SLA.

> [!info] Illustrative example
> The scenario, company (`example.com`), timestamps, and revenue figures are
> fictional but structurally realistic. Names (`@maria`, `@sam`, `@jhonathan`)
> are placeholders. The technical failure mode (PG15 default password
> encryption change breaking pgbouncer md5 hashes) is real and worth
> internalizing on any actual PG major upgrade.

### Sample 1 — Maintenance announcement (posted T-72h)

```
🛠️ [SCHEDULED] api-db-01 — Postgres 14 → 15 major version upgrade

When:      2026-09-20 02:00–03:30 UTC  (90 min window)
Impact:    Checkout API, Orders API, Admin panel
Downtime:  Partial — ~5 min read-only window during promotion at 02:20 UTC
Reason:    PG14 EOL 2026-11; unlocks logical replication for CHG-2026-0920-001
Owner:     @maria  |  Backup: @sam
Runbook:   https://wiki/runbooks/pg-major-upgrade
Status:    https://status.example.com

Rollback:  Failover to PG14 replica (kept in sync via pg_upgrade --link mode).
           Trigger: >2 min error rate above 1% on Checkout API.
           ETA to restore: 8 min.
```

### Sample 2 — Change request (approved 48h before window)

```
Change ID:   CHG-2026-0920-001
Title:       Upgrade api-db-01 from Postgres 14.11 → 15.7
Type:        Normal
Risk:        Medium
Priority:    Medium

── Context ──
What:  In-place major-version upgrade of prod primary via pg_upgrade in --link mode.
       Replica api-db-01-replica stays on 14.11 as rollback target until T+48h burn-in.
Why:   PG14 goes EOL 2026-11-12 (vendor advisory). PG15 also unlocks logical
       replication features required by the analytics pipeline (see ADR-0031).
Systems:  api-db-01 (prod primary), Checkout API, Orders API, Admin panel
Users:    All customers during 5-min read-only window; internal ops during full window

── Plan ──
Window:  2026-09-20 02:00–03:30 UTC  (low-traffic Sun night)
Steps:
  1. 02:00 — Announce start in #maintenance — @sam
  2. 02:05 — Stop writes, drain connection pool — @maria — 3 min
  3. 02:08 — Snapshot RDS + verify — @maria — 5 min
  4. 02:13 — Run pg_upgrade --link — @maria — 7 min
  5. 02:20 — Promote, restart pgbouncer, resume writes — @maria — 2 min
  6. 02:22 — Smoke tests: checkout, order create, admin login — @sam — 10 min
  7. 02:35 — Monitor for 45 min — @maria + @sam
Validation:
  - Checkout API p95 < 400ms (baseline: 320ms)
  - Zero 5xx on synthetic monitor for 30 min
  - Replica lag < 2s
  - `SELECT version();` reports 15.7

── Risk ──
Blast radius:   Full checkout outage if promotion fails or app pool can't reconnect.
                Revenue impact ~€800/min based on Sunday 02:00 UTC baseline.
Rollback plan:  1. Stop writes on PG15 primary.
                2. Promote api-db-01-replica (still on 14.11).
                3. Update pgbouncer target, reload.
                4. Restart app pods to flush connections.
                ETA: 8 min. Decision trigger: >2 min at >1% 5xx.
Rollback owner: @maria
Dependencies:   No other CHG in window. Release freeze active until 03:30 UTC.

── People ──
Implementer:  @maria
Approver:     @jhonathan  ✅ approved 2026-09-18 14:22 UTC
Reviewer:     @sam
Comms:        @sam

── Comms plan ──
T-72h: #maintenance announcement ✅ posted
T-24h: #maintenance reminder
T-0:   Start post in #maintenance
T+X:   Completion or rollback post
```

### Sample 3 — Outage (cutover fails)

```
Incident ID:  INC-2026-0920-001
Title:        Checkout API 100% 5xx after PG15 cutover — connection refused
Severity:     SEV1
Started:      2026-09-20 02:22 UTC  (detected: 02:23 UTC)
Detected by:  Synthetic checkout monitor + PagerDuty
Status:       Resolved

── Impact ──
Users:        All checkout traffic globally (~1,200 attempted transactions)
Services:     Checkout API, Orders API (Admin panel unaffected — different pool)
Business:     ~€38k in delayed revenue; SLA breach on Enterprise tier (99.95% monthly)
Security:     No

── Channels ──
War room:     #inc-2026-0920-001
Bridge:       Not opened — all responders in Slack
Status page:  https://status.example.com — updated 02:26 UTC
Related CHG:  CHG-2026-0920-001
```

**Timeline (UTC):**

```
02:22 — pg_upgrade completes, apps begin reconnecting to PG15
02:23 — Checkout synthetic monitor fires SEV2; @maria pages
02:24 — @maria escalates to SEV1: 100% 5xx, not partial degradation
02:25 — @jhonathan assumes IC; @sam Comms Lead; @maria Ops Lead
02:26 — Status page: "Investigating checkout failures"
02:28 — Root cause hypothesis: pgbouncer "SCRAM authentication failed"
02:33 — Confirmed: PG15 changed default password_encryption to scram-sha-256;
        pgbouncer userlist.txt still had md5 hashes
02:38 — Decision point: fix forward (regenerate hashes) or roll back
02:39 — IC calls fix-forward — rollback 8 min, fix ~5 min, both risky
02:44 — New userlist.txt deployed, pgbouncer reloaded
02:46 — Checkout API recovering, error rate dropping through 30%
02:52 — Error rate at baseline (<0.1%); synthetic checks green
03:09 — 15-min stable window confirmed; incident resolved
03:12 — Status page: resolved
```

**Resolution:**

```
Resolved at:    2026-09-20 03:09 UTC
Total duration: 47 min  (detect → resolve)
TTR breakdown:  Detect: 1 min | Diagnose: 10 min | Fix: 6 min | Verify: 30 min
Root cause:     PG15 default password_encryption changed from md5 to scram-sha-256.
                Runbook didn't include regenerating pgbouncer userlist.txt hashes.
Mitigation:     Regenerated hashes, reloaded pgbouncer.
Fix:            Same — permanent.
Verification:   Synthetic checkout monitor green for 30 min; p95 at baseline.

Follow-ups filed:
- [ ] TICK-4412 — Update pg-major-upgrade runbook w/ pgbouncer auth step — @maria — 2026-09-22
- [ ] TICK-4413 — Add pgbouncer-auth smoke test to CHG validation — @sam — 2026-09-24
- [ ] TICK-4414 — Postmortem draft — @jhonathan — 2026-09-25
```

**Postmortem excerpt:**

```
── What went well ──
- Detection was 1 min from cutover — synthetic monitor did its job
- IC/Ops split held; @maria never got pulled into coordination
- Rollback plan existed and was seriously considered, not just documented

── What went poorly ──
- Runbook was written for PG13→14. PG14→15 path wasn't tested against our
  specific pgbouncer config.
- Dry-run environment doesn't mirror pgbouncer setup.

── Where we got lucky ──
- Sunday 02:00 UTC is our lowest-traffic window — same failure at 14:00 UTC
  Monday would have been ~15x the revenue impact.
- pgbouncer error was loud and specific. Silent degradation would have made
  TTR 2-3x longer.

── Action items ──
| # | Action | Owner | Priority | Due |
|---|--------|-------|----------|-----|
| 1 | Fix pg-major-upgrade runbook (scram step) | @maria | P0 | 2026-09-22 |
| 2 | Bring staging pgbouncer config in line with prod | @sam | P1 | 2026-10-05 |
| 3 | Add "auth mechanism changes" to CHG validation checklist | @sam | P1 | 2026-09-24 |
| 4 | Evaluate moving to native PG pooling (pgcat) | @maria | P2 | 2026-11-01 |

── Lessons ──
- Major version upgrades change defaults, not just features. Runbook framing
  ("upgrade steps") missed this — future runbooks need explicit "default
  changes to audit" section pulled from vendor release notes.
- "Staging mirrors prod" is a claim to verify, not assume. Pgbouncer config
  drift was the enabling condition here.
```

---

## Gotchas

- **Don't force these formats onto a channel that has its own norms.**
  For the internal cybersoft `! IMP - Outages` channel, use
  [[wiki/runbooks/outage-channel-announcement]] instead. Adding SEV labels
  or ITIL scaffolding to a channel that doesn't use them looks like process
  bloat.
- **CHG type discipline matters.** If everything gets marked "Standard,"
  the process degenerates to a rubber stamp. If everything gets marked
  "Normal," you burn approver cycles. Reserve Standard for genuinely
  repeatable, pre-approved procedures.
- **The postmortem "Where we got lucky" section is the most-skipped and
  most-valuable section.** Force yourself to write it. It's where the
  next incident's investment case lives.
- **`Related CHG:` on the outage record is what closes the learning loop.**
  Without it, postmortem action items land as generic "be more careful"
  instead of concrete "update CHG-XXXX checklist step N."

## Related

- [[wiki/runbooks/outage-channel-announcement]] — cybersoft `! IMP - Outages`
  channel-specific variant (STATUS-tagged, HTML `<br>` combined postmortem)
- [[wiki/runbooks/gitlab-runner-token-rotation]] — example of a specific
  operational runbook that a CHG record would reference
- [[wiki/learnings/2026-09-12-hpa-memory-requests-pin-max]] — example of a
  learning that came out of a real postmortem
