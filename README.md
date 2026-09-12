# 🧠 Kephaleos

**Personal & professional AI-powered Second Brain.**

Portable, local-first, Markdown-based. Inspired by
[Andrej Karpathy's LLM Wiki concept](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

---

## The idea

```
EXPERIENCE → REMEMBER → KEPHALEOS → CONNECT → RECALL → APPLY → LEARN → IMPROVE
```

Kephaleos is not a note-taking app. It is a compounding knowledge system.
Raw sources go in. An LLM (Claude) compiles them into a canonical wiki.
When you need to remember something later, you ask the wiki — not your past
self.

## The stack (deliberately boring)

| Layer | Tool | Replaceable? |
|-------|------|--------------|
| Storage | Markdown files | ✅ |
| History | Git | ✅ |
| Human UI | Obsidian | ✅ |
| Intelligence | Claude Code | ✅ |
| The brain | **Kephaleos** | ❌ (it's your knowledge) |

If any layer disappears, the knowledge survives.

## Layout

```
~/kephaleos/
├── 00-dashboard/     nav
├── raw/              immutable source material
├── wiki/             compiled canonical knowledge (the brain)
├── work/             active operational context
├── personal/         personal planning & journal
├── templates/        page skeletons
├── archive/          demoted knowledge
├── CLAUDE.md         Claude's operating charter
├── index.md          knowledge map
└── log.md            append-only activity log
```

Read [`CLAUDE.md`](./CLAUDE.md) for the full architecture and rules.

## Daily use

From **any** directory (not just this one):

```
Remember this in Kephaleos:
  <thing worth preserving>
```

```
Recall from Kephaleos:
  <question about what you already know>
```

Or use the slash commands available globally in Claude Code:

- `/kephaleos-remember` — preserve a lesson, decision, solution, or context
- `/kephaleos-recall` — query the accumulated knowledge
- `/kephaleos-decision` — record a decision with full ADR-style context
- `/kephaleos-review` — weekly synthesis pass
- `/kephaleos-lint` — health check

## The value filter

> Would future-me be annoyed if I had to figure this out again?

If no, don't store it. Signal over volume.

## What Kephaleos will refuse to do

- Store secrets, tokens, or credentials (auto-redacted)
- Create endless duplicate pages for the same concept
- Fabricate history you don't actually have
- Present synthesis as source fact

## Backup

`~/kephaleos/` is a git repo. Add a private remote when you're ready:

```bash
cd ~/kephaleos
git remote add origin git@github.com:<you>/kephaleos.git
git push -u origin main
```

Do not push to a public repo.
