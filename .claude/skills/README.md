# Vault-scoped Kephaleos skills

The **primary** Kephaleos interface is the set of global slash commands
installed in `~/.claude/commands/`:

- `/kephaleos-remember`
- `/kephaleos-recall`
- `/kephaleos-decision`
- `/kephaleos-review`
- `/kephaleos-lint`
- `/kephaleos-capture`
- `/kephaleos-ingest`
- `/kephaleos-meeting`

These work from **any** directory — you do not need to `cd ~/kephaleos`.

This directory (`~/kephaleos/.claude/skills/`) is reserved for
**vault-scoped** skills added in Phase II or later — specialized behaviors
that only make sense when Claude is actively working inside the vault
(e.g. bulk re-compilation, taxonomy migrations, custom link resolvers).

Phase I intentionally ships nothing here. The knowledge repository stays
portable: nothing under Kephaleos requires Claude Code to be useful.
