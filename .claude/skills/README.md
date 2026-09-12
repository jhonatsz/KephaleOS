# Vault-scoped Kephaleos skills

The **primary** Kephaleos interface is the set of global slash commands
installed in `~/.claude/commands/` (unchanged in Phase 1.1):

- `/kephaleos-remember` — improves existing canonical pages first
- `/kephaleos-recall` — evidence-ranked; personal experience prioritized
- `/kephaleos-decision` — ADR-style, alternatives + consequences required
- `/kephaleos-capture` — lightweight unprocessed capture to inbox
- `/kephaleos-ingest` — compile inbox items into canonical wiki
- `/kephaleos-meeting` — file a meeting; extract durable takeaways
- `/kephaleos-review` — weekly signal pass (patterns, not counts)
- `/kephaleos-lint` — quality checks; no destructive auto-fixes

All work from **any** directory. You do not need to `cd ~/kephaleos`.

This directory (`~/kephaleos/.claude/skills/`) is reserved for
**vault-scoped** skills added in Phase II or later — specialized behaviors
that only make sense when Claude is actively working inside the vault
(bulk re-compilation, taxonomy migrations, custom link resolvers, etc.).

Phase 1.1 intentionally ships nothing here. The knowledge repository
stays portable: nothing under Kephaleos requires Claude Code to be useful.
