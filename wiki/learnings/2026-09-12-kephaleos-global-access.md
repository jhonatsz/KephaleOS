---
type: learning
status: active
created: 2026-09-12
updated: 2026-09-12
tags: [kephaleos, meta, workflow, claude-code]
sources: []
confidence: high
---

# Kephaleos must be accessible from any working directory

## Context

Kephaleos is a personal Second Brain. In daily use you work inside many
different repos — infrastructure, application projects, homelab. Requiring
`cd ~/kephaleos && claude` before every remember/recall interaction would
add enough friction to guarantee the system doesn't get used.

## What happened

Phase I bootstrap test: capture a lesson while working from `/tmp` (a
directory that has nothing to do with the vault), then recall it from a
different directory. If either step required changing into `~/kephaleos`,
the whole architecture would have failed the friction test.

## The lesson

**Kephaleos must be accessible without launching Claude from the Kephaleos
repository.** The knowledge lives at `~/kephaleos/`, but the interface is
global. Two mechanisms make this work:

1. **Global slash commands** in `~/.claude/commands/kephaleos-*.md` are
   available in every Claude Code session regardless of cwd. They operate
   on `~/kephaleos/` via absolute paths.
2. **Global CLAUDE.md router** at `~/.claude/CLAUDE.md` teaches Claude to
   route natural-language phrases ("Remember this in Kephaleos", "Recall
   from Kephaleos") to the right commands from any working directory.

## Why it matters

- Reduces friction to near-zero. If capture takes a `cd`, it won't happen.
- Preserves the single-source-of-truth invariant: knowledge lives at
  `~/kephaleos/`, is never copied into other repos, and other repos never
  need to know Kephaleos exists.
- Guards against a common failure mode: per-project brains that fragment
  the knowledge and can't synthesize across projects.

## Anti-patterns this rejects

- ❌ Copying the vault into `~/.claude/` (breaks portability, mixes
  configuration with knowledge).
- ❌ Making each project have its own `.kephaleos/` (fragmentation).
- ❌ A CLI wrapper like `kephaleos add "…"` (extra dependency, breaks the
  "if Claude disappears, the vault still works" invariant).

## Related

- [[CLAUDE.md]] — the Kephaleos knowledge-maintainer charter
- [[README.md]] — architectural stance on portability

## Sources

- Phase I bootstrap, 2026-09-12 (this note is itself the test artifact)
