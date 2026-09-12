---
type: dashboard
updated: 2026-09-13
---

# Today

What deserves attention now. Refresh — don't append endlessly. Git keeps history.

External systems (Jira, GitHub, Linear, calendar) own tasks. This view
summarizes; it does not duplicate them.

---

## Top priorities

- File IT ticket to have Zscaler add the FortiGate IP to the ZPA bypass
  list. Durable fix for the VPN-over-VPN routing conflict — see
  [[Solution: VPN fails when another VPN/ZTNA agent installed]].
- Migrate corp tooling (Zscaler + FortiClient) off personal Mac onto
  work-issued machine when it arrives.

## Active projects

- **Kephaleos Phase 1.1** — knowledge-quality upgrade in progress
  (this session).

## Open decisions

*(none recorded — `wiki/decisions/` is empty. Highest-value knowledge
type has zero entries — worth capturing one this week.)*

## Follow-ups

Open, from [[Incident: GitLab Runner CPU reservation]]:

- Raise `sbiqai` `requests.memory` 256Mi → 512Mi (both namespaces)
- Lower `sbiqai` `requests.cpu` 250m → 50m
- Reconsider memory-HPA for `sbiqai` (steady-state workload)
- Add alert for pods in `Pending` > 5 min
- Extend right-sizing review to `tasktile`, `cybersoft-dtr`, `adminsynonyms`

## Recent important knowledge

- [[HPA memory-requests pin at max]] — new learning, 2026-09-12
- [[Solution: VPN fails when another VPN/ZTNA agent installed]] — new
  solution, 2026-09-12

## Unresolved problems

- Zscaler / FortiClient routing conflict — temporary route-override
  workaround in use; dies on reboot/Wi-Fi change/Zscaler policy refresh.

## Knowledge gaps

See [[knowledge-map]] · *Current knowledge gaps* section.

## Inbox

- `raw/inbox/` — currently empty (`.gitkeep` only). Nothing to compile.
