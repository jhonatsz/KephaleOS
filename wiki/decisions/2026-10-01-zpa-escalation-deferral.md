---
type: decision
status: accepted
created: 2026-10-01
updated: 2026-10-02
tags: [zscaler, zpa, vpn, networking, macos, process]
sources:
  - "[[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]]"
  - "[[raw/notes/2026-09-15-zpa-hijack-partner-sftp]]"
  - "[[raw/notes/2026-09-22-zpa-hijack-ec2-runner-ssh]]"
  - "[[raw/notes/2026-10-01-0456-zpa-route-command]]"
  - "[[raw/notes/2026-10-01-teams-offline-isp-switch-stale-ne]]"
supersedes: []
superseded-by: []
---

# Decision: Defer the ZPA policy scope review; keep using local fixes

## Status

accepted 2026-10-01 — carried forward from the Zscaler hijack solution page's §"In practice" note where the deferral was first captured. First retrospective decision formally recorded in `wiki/decisions/`.

## Context

Between 2026-09-12 and 2026-10-01, the operator's personal Mac was hit by **four Zscaler Client Connector interference incidents** against unrelated destinations:

| # | Date | Destination | Mode | Shape |
| --- | --- | --- | --- | --- |
| 1 | 2026-09-12 | Corp FortiGate (IPsec) | A — FIB hijack | UDP IKE timeout |
| 2 | 2026-09-15 | Partner SFTP (TCP 2233) | A — FIB hijack | Fast TCP RST |
| 3 | 2026-09-22 | Employer AWS EC2 CI runner (SSH 22) | A — FIB hijack | Silent timeout + ICMP |
| 4 | 2026-10-01 | MS Teams signaling (52.123/16) | **B — stale NE** | `EADDRNOTAVAIL` after ISP switch |

[[Zscaler ZPA route hijack]] carries the technical "third-occurrence rule" — once this pattern hits a third unrelated destination, the recommendation is to stop filing per-host bypass tickets and ask IT for a **ZPA policy scope review** (the ZPA Application Segment list is likely over-broad, catching large public-internet ranges with no business justification).

By 2026-09-22 the third-occurrence rule had triggered. The operator chose not to pursue the IT escalation. On 2026-10-01 (morning), the operator stated the preference plainly:

> "for ZPA i just stick to route command to fix it"

On 2026-10-01 (evening), incident #4 landed with a **new failure mode** (Mode B — stale NetworkExtension socket-claim) whose fix is not the `route` command but `sudo pkill -HUP ZscalerTunnel`. The operator applied the fix and continued without escalating.

This decision records the deferral explicitly rather than leaving it as a floating note on the solution page, so future-self can revisit the reasoning when the ranking signals flip.

## Decision

**Do not open the ZPA policy scope review ticket with IT at this time.** Continue using the two local quick-fixes (route override for Mode A; `pkill -HUP ZscalerTunnel` for Mode B) each time the hijack fires. Revisit this decision when any of the specific trigger conditions below are observed.

## Why

- **Friction ranking.** A 30-second local command beats scheduling a policy-scope conversation with IT. Even at 4 incidents in 20 days, the per-incident cost is lower than the one-time escalation cost (ticket creation, back-and-forth with IT, scope-review scheduling, waiting on policy change, verification).
- **Low-visibility blast radius.** All four incidents affected only the operator's personal workstation. None caused missed meetings, blocked customer-facing work, or triggered a downstream outage. The friction is personal, not organizational, so the escalation cost is paid by the operator alone with no cross-team amplification to offset it.
- **The operator is already migrating.** The documented plan is to move corp tooling (Zscaler + FortiClient) off the personal Mac onto a work-issued machine. That migration — not an IT policy change — is the structural fix. An IT escalation now is work on infrastructure that is already slated for removal from this host.
- **The failure modes are becoming cataloged, not just worked around.** Four incidents produced two distinct enforcement-layer fingerprints with mode-specific diagnostics and fixes. The canonical solution page is now a decision tool, not just a workaround ledger. Each future recurrence takes <1 minute of diagnostic work.
- **ZPA policy decisions are IT's territory.** Even if the escalation went well, the resulting policy change is one the operator doesn't control; another broad-net policy push from the EMS console in the future would re-break everything. The structural fix (migrate off personal Mac) removes the dependency on IT discipline altogether.

## Alternatives considered

- **Open the ZPA policy scope review ticket now.** Rejected. The one-time cost outweighs continued per-incident friction *at the current recurrence rate and blast radius*. If either rises (see Risks below), this becomes the right path.
- **Request a one-off per-host bypass for each incident.** Already how the third-occurrence rule says to stop operating — four per-host tickets would be noise with no systemic improvement. Rejected.
- **Uninstall FortiClient entirely to simplify the stack.** Would eliminate FortiClient's latent NE risk but doesn't address the actual culprit in incidents #1–#4 (Zscaler). Also blocked by employer admin passcode. Not a substitute for this decision; parallel hygiene action.
- **Accelerate the work-laptop migration to make this moot.** The right long-term answer, but requires an IT ticket + hardware provisioning + corp-tool re-provisioning on new hardware — longer wall-clock than this decision horizon. Still the stated plan; this decision is the bridge until that lands.

## Consequences

**Easier**
- Zero wait on IT for the active workaround path.
- No ownership of a policy-scope conversation the operator doesn't fully control the outcome of.
- Keeps the operator in a diagnostic/fix cadence that reinforces the two-mode mental model (useful if the same pattern shows up on the work-issued laptop later).

**Harder**
- Incident #5, #6, #N will land. Each will cost a minute of diagnostic work and 5–10 seconds of fix time. Cumulatively significant if the base rate rises.
- Mode B's trigger surface is broader than Mode A's — "any ISP/Wi-Fi switch" fires it, not just "communicate with destination X". On travel days or tethering, the recurrence rate will likely be higher than the Mode A base rate observed so far.
- The ZPA policy stays over-broad for everyone else on the same policy, not just the operator. This decision is personal; others on the same policy who don't know the local fixes are paying the full friction cost silently.

**Locked in (until explicit revisit)**
- Operator owns the diagnostic + fix flow for every ZPA hijack / stale-NE event on this host until the work-laptop migration completes or a trigger below fires.

## Risks

**Signals that would trigger a revisit of this decision:**

1. **Recurrence rate climbs past ~1/week sustained for 2+ weeks.** The friction-ranking math inverts around that cadence.
2. **A hijack affects a shared or non-macOS-routable destination** — one the local `route` command can't fix (e.g., a destination whose IP can't be pinned to a LAN gateway, or a resource shared with others who are blocked by the same policy).
3. **A hijack causes a user-facing miss** — missed customer meeting, late incident response, blocked PR review window. First time the friction leaves the operator's own machine and reaches the organization, the ranking flips immediately.
4. **The work-laptop migration slips past 2026-12-31** — if the structural fix is 90+ days away, re-run the math on the escalation vs. deferred-friction tradeoff with that longer tail.
5. **Zscaler EMS pushes a policy change that broadens scope further** — if a new mystery destination starts failing, assume this and escalate the policy-scope conversation then rather than per-host.

## Related

- [[Zscaler ZPA route hijack]] — the canonical technical solution page carrying the mode taxonomy and quick-fixes
- [[00-dashboard/knowledge-map]] — gap entry *"ZPA policy scope review with IT — deferral under pressure"* mirrors this decision
- Local memory file `~/.claude/projects/-Users-jhonatsz/memory/project_corp_vpn_on_personal_mac.md` carries the identical deferral record for Claude Code sessions outside the vault (parallel source of truth; keep in sync when either changes)

## Sources

See frontmatter — five raw notes spanning the four incidents and the morning operator-practice capture.
