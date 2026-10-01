---
type: decision
status: accepted
created: 2026-10-02
updated: 2026-10-02
tags: [kephaleos, meta, phase-gate, process]
sources:
  - "[[CLAUDE.md]]"
  - "[[raw/inbox/2026-10-02-0152-kephaleos-improvement-asks]]"
supersedes: []
superseded-by: []
---

# Decision: Phase II entry criteria for Kephaleos

## Status

accepted 2026-10-02 — forward-looking decision, no features adopted today. Serves as the ladder for allowing any Phase 1.1-forbidden feature in the future. Vault's third ADR; first prospective (vs retrospective) decision captured.

## Context

Charter §28 explicitly forbids Phase 1.1 from introducing:

- Vector DBs and embeddings infrastructure
- RAG servers
- Autonomous multi-agent teams
- Slack / email / Drive / GitHub / calendar ingestion
- Scheduled autonomous ingestion
- Complex Obsidian plugin stacks
- Local model infrastructure
- Databases
- Custom web dashboards

The charter rationale is sound — Phase 1.1 should prove that **operator discipline on plain markdown + git** can produce useful knowledge compression before adding machinery. But the charter gives **zero entry criteria** for when these features become allowed. In practice this means:

1. Discipline failures (e.g., `today.md` 18 days stale on 2026-10-02) have no defined response path — just "do better"
2. Any future adoption decision will be made **under pressure** (friction has already built up) rather than **under design** (vivid memory of what the pain actually was)
3. Scope creep risk is high — once one Phase II feature lands, there's no articulated ladder for what should follow

Writing this ADR now, while Phase 1.1 pains are **vivid** (an 18-day-stale dashboard, a 17-day-idle inbox batch, 4 Zscaler incidents producing a 3-page synthesis), is the cheapest moment to decide the ladder.

## Decision

**Define concrete, measurable entry criteria for each Phase II feature class.** Keep features out of the vault until a specific trigger fires. Order Phase II adoption by risk + ROI, not by novelty. Carry forward hard limits from charter §7 (secrets), §9 (provenance), §10 (contradictions), §27 (portability) that persist regardless of phase.

The entry criteria below act as **triggers**, not **green lights**. A trigger firing opens the design conversation; it doesn't auto-approve adoption. Each adoption is a separate decision.

## Entry triggers (concrete, measurable)

| Feature class | Entry trigger | Order within Phase II |
| --- | --- | --- |
| **Embeddings-for-search** (upgrade `/kephaleos-recall` with semantic match) | 3+ instances in a month of "I know this is in Kephaleos but can't find it via grep", OR canonical pages exceed 300 | **1 (cheapest, highest ROI)** |
| **GitHub ingestion** (auto-capture PR activity, issue state) | Manual sync between `/daily-brief` output and Kephaleos exceeds 30 min/week for 2 weeks | 2 |
| **Scheduled autonomous ingest** (cron-ish `/kephaleos-ingest`) | Average unprocessed inbox age exceeds 14 days for 2 consecutive weeks | 3 |
| **RAG for synthesis queries** (multi-page semantic retrieval) | Repeated recall queries that grep can't satisfy — needs semantic match across 3+ pages. Depends on embeddings already landed | 4 |
| **Calendar / Slack / email ingestion** | Specific recurring use case where manual capture costs >1h/week and the signal rate is high enough to justify the noise | 5 |
| **Local model infrastructure** | External LLM cost exceeds $50/mo on vault operations, OR a specific privacy requirement blocks cloud inference | 6 |
| **Database projection** (read-only index over markdown) | Markdown + git becomes slow enough that `/kephaleos-recall` latency exceeds 10 s on typical queries | 7 (Phase II edge) |
| **Custom web dashboard** | External stakeholder needs read-only access AND Obsidian Publish is insufficient | 8 (Phase II edge; most Phase III) |
| **Autonomous multi-agent teams** | **No Phase II trigger.** Phase III territory. Autonomous writes to `wiki/` are a §25 "user corrections" violation by construction | — |

## Why these specific triggers

- **Embeddings first** because it upgrades the recall axis without changing storage, data flow, or trust model — reversible, additive, and the vault stays usable without it. Lowest-risk Phase II feature.
- **GitHub next** because `/daily-brief` already runs against GitHub, so the operator-system touchpoint is defined. Adopting one external connector is easier than designing an N-connector architecture.
- **Scheduled ingest after** the above two, because once embeddings make retrieval cheap and GitHub proves the external-system pattern, scheduling becomes a smaller leap. The trigger (14-day inbox average) is observable from `log.md` without instrumentation.
- **RAG waits for embeddings** by construction — no point adopting RAG infrastructure until the retrieval primitive it depends on is stable.
- **Local models last within Phase II** because they're cost/privacy driven, not feature driven. Deciding "we need a local model" before deciding "we need embeddings" is backwards.
- **Autonomous agents are Phase III**, not late Phase II. Letting an agent write to `wiki/` without human approval violates §25 ("user corrections are high-priority evidence; do not blindly delete historical evidence") — the review step is load-bearing, not ceremonial.

## Hard limits (persist across all phases)

These do not relax regardless of phase:

- **Portability (§27).** The vault must survive Obsidian disappearing *and* Claude being replaced. No feature may make either invariant false. Markdown + git stay source of truth.
- **No external secret stores (§7).** Secrets never land in the vault, no exceptions, no feature-flag opt-ins. Local Claude Code memory remains the authorized channel for sensitive org/project context.
- **No lossy auto-summarization that replaces raw notes (§anti-pattern).** Compilation into `wiki/` is additive; the raw evidence in `raw/` is immutable during normal ingest.
- **No auto-overwrite of contradictory evidence (§10).** Automated systems must surface conflicts, not resolve them silently.
- **No proprietary platform binding** that creates a non-reversible migration path (e.g., a hosted RAG product that owns the embeddings and won't export them).
- **No autonomous writes to `wiki/`** without explicit human approval of the diff. Phase III may introduce agent-assisted writes with required review; it must never be reversed.

## Alternatives considered

- **Leave the charter as-is and decide ad-hoc when a trigger fires.** Rejected. Deciding under friction produces worse outcomes than deciding under design. The 18-day-stale dashboard is evidence that "we'll decide later" quietly becomes "we never decided and operator burnout chose for us."
- **Pre-approve a specific Phase II feature now** (e.g., embeddings) because the trigger is clearly close. Rejected. The point of this ADR is the ladder, not the next step. Pre-approval collapses two decisions into one and loses the forcing function of "does the trigger actually fire?"
- **Define triggers only, no ordering.** Rejected. Order matters because dependencies exist (RAG needs embeddings; scheduled ingest benefits from embeddings; connectors benefit from a stable retrieval primitive). The ordering is itself part of the decision.
- **Redefine the phase gate itself** (e.g., merge Phase II into 1.1). Rejected. The gate is doing useful work — forcing discipline-first. Collapsing it would surrender the forcing function. If anything, the gate should be *tighter*, not looser.

## Consequences

**Easier**
- Any future adoption conversation starts from a defined ladder, not a blank page
- Discipline failures (stale dashboards, aging inbox) have a defined response path: *is a trigger firing? If so, enter the Phase II conversation for that specific feature. If not, discipline is the correct response*
- Scope creep is harder — no "oh and while we're adding embeddings, let's also add Slack" bundles
- Trigger values are measurable from existing vault artifacts (`log.md`, inbox file timestamps, page counts, `/kephaleos-recall` miss notes) without new instrumentation

**Harder**
- Operator pressure (e.g., "I just want vector search now") now has to argue against a written ladder, not a vague "later"
- Features that are *useful* but don't match any trigger (hypothetical: a nice-to-have viz tool) have no path in — the ladder is deliberately not exhaustive. New feature classes require amending this ADR

**Locked in (until explicit revisit)**
- Autonomous multi-agent writes to `wiki/` stay forbidden through Phase II regardless of other triggers
- Secrets stay out of the vault, period
- Portability (§27) stays non-negotiable

## Risks

**Signals that would trigger revisiting this decision:**

1. **Any trigger fires but the Phase II adoption is still blocked by this ADR's ordering.** If embeddings' trigger hasn't fired yet but RAG's has, this ADR says to adopt embeddings first anyway — if that's wrong in a specific case, revisit.
2. **A new feature class emerges** that doesn't fit any row in the trigger table (plausible candidates: on-device AR/VR knowledge surfaces, voice capture, mobile companion). Amend the table; don't improvise.
3. **A trigger measurement proves unmeasurable in practice.** The "3+ recall misses per month" trigger assumes the operator notices and records misses. If this doesn't happen reliably, the trigger is noise and needs replacement with something observable.
4. **Phase III design begins.** Phase III (autonomous agents, external stakeholder surfaces, multi-user) requires a successor ADR; this one won't absorb that scope.
5. **Charter §28 itself changes.** If the charter's forbidden list expands or contracts, re-align this ADR to match.

## Revisit cadence

- **On any trigger firing** — open the specific Phase II feature's design conversation
- **Quarterly** — scan the ADR, update trigger values if operator behavior has shifted
- **On every charter §28 change** — re-align to the charter

## Related

- `~/kephaleos/CLAUDE.md` §28 — the phase gate this ADR interprets
- [[wiki/decisions/2026-10-01-zpa-escalation-deferral]] — the vault's first retrospective ADR; this one is the first *prospective* ADR, same template
- [[raw/inbox/2026-10-02-0152-kephaleos-improvement-asks]] — inbox note that drove this (improvement #5 of the top-5)
- `~/kephaleos/00-dashboard/knowledge-map.md` — gap "Phase II trigger criteria undefined" closed by this ADR

## Sources

See frontmatter.
