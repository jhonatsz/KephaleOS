---
type: project
status: active
created: 2026-09-15
updated: 2026-09-15
aliases:
  - cybersoftbpo.com rebuild
  - CyberSoft website
  - CyberSoft Payload rebuild
tags: [nextjs, payload-cms, postgres, s3, docker, tailwind, wix-migration, cybersoft]
confidence: high
repo: /Users/jhonatsz/workspace/cybersoft/cybersoftbpo.com (local, not yet pushed)
---

# CyberSoft public website rebuild

Replaces the Wix-hosted `cybersoftbpo.com` marketing site with a
self-hosted Next.js 15 + Payload CMS 3 stack, giving non-technical
staff a full admin without vendor lock-in. Belongs to
[[wiki/organizations/cybersoft]].

## Purpose

- Remove Wix vendor lock-in (design and content ownership)
- Give editors a CMS-backed admin (drafts, versioning, media library,
  RBAC) so copy/images can be changed without touching code
- Foundation for future dynamic sections (blog, service/industry
  detail pages, forms integrated with internal systems) that Wix would
  not support cleanly

## Status (2026-09-15)

- **Working locally** at `http://localhost:3010`; admin at
  `/admin` (default seed user `admin@cybersoftbpo.com` — rotate on
  first login).
- **Milestones M1–M7 landed:** Next+Payload skeleton, tokens+fonts,
  routes, scrape, contact form (Turnstile + SMTP), admin polish,
  Playwright smoke suite + Caddy prod compose.
- **Home page visually mirrors live** (skyline hero + cream overlay
  panel + two-part red CTA + navy Our Services with cream cards +
  navy Contact Us + navy footer).
- **Other public pages** (`/about`, `/industries`, `/platforms`,
  `/contact`, `/policies/*`) still show the flat scraped rich-text
  dump inside the new layout system — each needs a hand-crafted block
  layout to match the live design.
- **Not yet pushed** to a remote or deployed.

## Architecture (durable knowledge)

Runs as one Next.js app that embeds Payload — same process serves
public site and admin.

| Layer | Choice |
|---|---|
| Framework | Next.js **15.4.11** (App Router, RSC) — pinned; Payload 3.89's peer range excludes 15.5 |
| CMS | Payload CMS 3 (React admin at `/admin`, REST at `/api/*`) |
| DB | Postgres 16 via `@payloadcms/db-postgres` |
| Media | S3-compatible object store via `@payloadcms/storage-s3` — MinIO for dev, S3/R2 in prod |
| Editor | Lexical rich-text |
| Email | `@payloadcms/email-nodemailer` (SMTP URL — Mailhog dev, Resend/Postmark prod) |
| Anti-spam | Cloudflare Turnstile (env-guarded — bypassed when key unset) |
| Styling | Tailwind CSS v4 + custom tokens |
| Fonts | `next/font` for Inter (body) + Spinnaker (heading) + Merriweather |
| Testing | Playwright smoke suite |
| Prod deploy | Docker Compose + Caddy TLS sidecar |

### Content model

Composable page blocks (Hero, RichText, ServicesGrid, FeatureList,
CtaBanner, ContactForm) applied to `pages` / `services` / `industries`
collections. Editors compose page layouts from the same blocks —
Gutenberg-style but strongly typed.

## Key decisions

- **CMS: Payload** (over Sanity, custom-from-scratch, MDX+Git). Reasons:
  Next.js-native, self-host friendly, built-in admin/auth/RBAC/media,
  no vendor per-editor pricing.
- **Deployment: self-host** (over Vercel+Neon or Vercel+SQLite).
  Reasons: data-residency control, full ownership, matches the
  existing CyberSoft infra posture (AWS ownership already in play — see
  [[wiki/projects/cybersoft-sftp]]).
- **Visual approach: pixel-perfect visual clone.** Extract real design
  tokens from the live DOM and reproduce section-by-section rather
  than doing a fresh design. Explicit user constraint.

## Design tokens (extracted from live site 2026-09-15)

Sampled via Playwright `getComputedStyle` on the live home page.

| Token | Value |
|---|---|
| `--color-brand` (navy) | `rgb(26, 43, 109)` |
| `--color-danger` (CTA red) | `rgb(212, 19, 23)` |
| `--color-cream` (headline panel / cards) | `#EAE5DA` |
| `--color-bg` | `#ffffff` |
| Heading font | Spinnaker (Google Fonts) |
| Body font | Wix-licensed `helvetica-w01-bold` (substituted with Inter + Arial fallback in the rebuild — licensed webfont is not transferable) |
| Content max-width | 1238px |
| Border radius | 0 (flat rectangles throughout) |
| Button radius | 0 |
| Hero aspect ratio | 21:9 (skyline photo) |
| Header height | 101px |

## Gotchas

### Wix scraping needs `waitUntil: 'domcontentloaded'`

Wix keeps analytics/websocket connections open indefinitely, so
Playwright's `networkidle` never fires and `page.goto` times out.
Use `domcontentloaded` + a 3–5s settle delay + a slow scroll pass to
force lazy-loaded imagery.

### Wix content is shadow-DOM-ish → seed the crawler with known paths

Nav links are JS-driven from a single mega-container. A DOM crawl
that treats top-level children as "strips" finds one giant strip and
nothing more. Fix: pre-seed the scraper's queue with the known site
paths (`/`, `/about`, `/industries`, `/platforms`, `/services`,
`/contact`, `/policies/*`) instead of trying to discover them by
crawling internal links.

### Local dev port collisions on macOS

The three default ports for this stack all collide with common
darwin/Docker Desktop artifacts:

| Service | Default | Collides with | Remapped to |
|---|---|---|---|
| Postgres | 5432 | Host Homebrew Postgres | **55432** |
| Next.js | 3000 | Grafana (via Docker on this machine) | **3010** |
| MinIO API | 9000 | Docker Desktop internal | **19010** |
| MinIO console | 9001 | — | **19011** |

`.env`, `docker-compose.yml`, and `package.json` scripts all reflect
the remap.

### Payload 3.89 peer conflict with Next 15.5

`@payloadcms/next@3.89.0` peer range excludes Next 15.5. Pin
`next` and `eslint-config-next` to `15.4.11`. Attempting `15.0.0`
resolves to 15.5.x by npm's default and installs fails
`ERESOLVE` unless `--legacy-peer-deps` (bad) — pin the exact version
instead.

### `importMap.js` auto-generates on first Payload boot

I initialize `src/app/(payload)/admin/importMap.js` as an empty stub
(`export const importMap = {}`); Payload rewrites it on first boot to
map every richtext feature and S3 client entry. That's expected — do
not commit the empty stub after the first run.

### tsx transpiles `page.evaluate` arrow fns → browser `__name` errors

Any `page.evaluate(() => {...})` passed through tsx gets esbuild's
`keepNames` treatment, injecting a `__name(fn, "name")` helper that
does not exist in the browser context. Symptom:
`ReferenceError: __name is not defined`. Fix: pass the evaluator body
as a raw string literal, not a TS arrow.

## Scripts

The workflow scripts under `scripts/` are the durable process
knowledge — each is idempotent and safe to re-run:

- `extract-design-tokens.ts` — sample live DOM, emit `src/styles/tokens.css`
- `scrape-live-site.ts` — Playwright crawl of live URLs, emit
  `tmp/content.json` + downloaded assets
- `import-scraped.ts` — upload scraped assets into Payload media,
  seed page layouts with heading + rich-text
- `deep-scan-home.ts` — inspect live home DOM for missing sections /
  parallax content
- `compare-screenshots.ts` — side-by-side live vs local screenshots
  per route and viewport width
- `seed-payload.ts` — canonical seed (admin user, section pages,
  service/industry collections, policy stubs)
- `seed-home-visual.ts` — hand-crafted home layout matching live
  section-by-section

## Lessons learned

- The scrape → import pipeline gets ~70% of the way. The last ~30%
  (section-by-section visual match) is manual per page — no scraper
  can infer the intended visual structure from Wix's compiled output.
- Extracting design tokens *first* and hand-building components
  against them (instead of eyeballing colors) produced a much closer
  visual match than a code-first approach would have.
- A dedicated `seed-{page}-visual.ts` per page (rather than one giant
  seed script) is the right unit — each page's visual structure gets
  its own hand-crafted layout that's easy to iterate on.

## Related

- [[wiki/organizations/cybersoft]] — parent organization
- [[wiki/projects/cybersoft-sftp]] — sibling CyberSoft infra project
