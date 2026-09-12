---
type: runbook
status: active
created: 2026-09-12
updated: 2026-09-13
aliases:
  - "Outage channel announcement runbook"
  - "! IMP - Outages format"
tags: [incident-communication, outage, runbook, cybersoft]
last-executed: 2026-09-12
---

# Runbook: Outage-channel announcement format

## When to use

Posting to the internal `! IMP - Outages` channel for any incident that
affects a shared system — even Sev-3 CI or infra issues with no
customer-facing impact. Consistency of format matters more than
completeness of any single post.

## Prerequisites

- Access to the outage channel
- Basic facts nailed down: what broke, when it started, blast radius, current status
- Owner assigned (usually yourself)

## Format — live incident post

```
<System>, <short qualifier>
🚨 Outage Alert — <one-line description>
STATUS: <MITIGATING | RESTORED | INVESTIGATING>

WHAT:
-- <bullet 1>
-- <bullet 2>

IMPACT:
-- <who is affected, what they can't do>
-- <what is NOT affected>

! IMP - Outages
```

`STATUS` values in use: `INVESTIGATING`, `MITIGATING`, `RESTORED`.
The `! IMP - Outages` line at the bottom is a channel-tag convention.

## Format — combined post (live incident + inline postmortem)

For Sev-3 incidents where a separate postmortem doc is overkill, append
the postmortem to the same channel post using a `<br>` separator (the
channel renders HTML):

```
<live incident post as above — usually STATUS: RESTORED>

! IMP - Outages

<br>

📝 Postmortem — <system> — <short title>

SUMMARY:
-- <2–3 bullets>

TIMELINE (times in <TZ>):
-- HH:MM — <event>
-- HH:MM — <event>

ROOT CAUSE:
-- <bullets>

ACTIONS TAKEN:
-- <bullets>

FOLLOW-UPS:
-- <bullets>

WHAT WENT WELL:
-- <bullets>

WHAT COULD BE BETTER:
-- <bullets>

OWNER: @<handle>
```

For Sev-1/2, post a link to a full postmortem doc instead of inlining.

## Steps

1. **First post — status = INVESTIGATING or MITIGATING.** Post as soon as
   the incident is confirmed and blast radius is known. Do not wait for
   root cause.
2. **Update in-thread.** Reply in the same thread as status changes
   (e.g., `INVESTIGATING → MITIGATING → RESTORED`). Don't create parallel
   top-level posts for the same incident.
3. **RESTORED post.** Once traffic / functionality is verified back to
   normal, edit or reply with `STATUS: RESTORED` and a one-line
   description of what mitigated it.
4. **Postmortem.** For Sev-3, append inline via `<br>` as above. For
   higher severities, link to the full doc.

## Verification

- Post is readable at a glance — the STATUS is visible without scrolling.
- Someone reading only the first three lines can tell if they are affected.
- No follow-up "who's affected?" or "when did this start?" DMs are needed.

## Gotchas

- **Don't use severity labels inside the format if the channel norm
  doesn't have them.** Adding a `Sev-3` line to a channel that doesn't
  use severity looks like process bloat.
- **Timestamps are optional in the template but expected in postmortems.**
  Fill in the TIMELINE section with real times from git commits, ticket
  activity, or Slack pings — not "around 2pm."
- **Root cause belongs in `WHAT` in the live post**, not a separate
  `CAUSE:` field — the template doesn't have one.
- **Don't tag @channel or @everyone** unless the incident is customer-facing
  and severity warrants a broadcast.

## Related

- [[Incident: GitLab Runner CPU reservation]] — first documented use of the combined-post format
