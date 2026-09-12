---
type: dashboard
updated: 2026-09-13
---

# Today

## Top priorities

- Open IT ticket: ask Zscaler admin to add the FortiGate IP to the ZPA
  bypass list — durable fix for the [[Solution: VPN fails when another VPN/ZTNA agent installed|VPN-over-VPN conflict]].
- Migrate corp tooling off personal Mac when the work-issued machine arrives.

## Follow-ups

Open from [[Incident: GitLab Runner CPU reservation]]:

- [ ] Raise `sbiqai` `requests.memory` 256Mi → 512Mi (both namespaces)
- [ ] Lower `sbiqai` `requests.cpu` 250m → 50m
- [ ] Reconsider memory-HPA for `sbiqai` (steady-state workload)
- [ ] Add `Pending`-pod > 5 min alert
- [ ] Extend right-sizing review to `tasktile`, `cybersoft-dtr`, `adminsynonyms`

## Recent important knowledge

- [[HPA memory-requests pin at max]] — new learning, 2026-09-12
- [[Solution: VPN fails when another VPN/ZTNA agent installed]] — new solution, 2026-09-12
- Phase 1.1 charter + commands upgrade, 2026-09-13

## Unresolved problems

- Zscaler / FortiClient routing conflict on personal Mac — temporary
  route-override in use; dies on reboot / Wi-Fi change / Zscaler policy refresh.

## Knowledge gaps

- **Decisions folder is empty.** Highest-value knowledge type has zero
  entries — worth capturing one this week via `/kephaleos-decision`.
- Zscaler bypass ticket outcome not yet captured.
- Proton VPN + FortiClient collision behavior — untested.
- Full gaps list: see [[knowledge-map]].

## Inbox

- `raw/inbox/` — empty. Nothing to compile.
