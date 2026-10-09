---
type: concept
status: active
created: 2026-10-10
updated: 2026-10-10
aliases:
  - DNS redirect
  - URL forwarding
  - Domain forwarding
  - CNAME at apex
  - Apex CNAME
  - 301 redirect vs DNS
tags: [dns, http, networking, redirect]
confidence: high
---

# DNS vs HTTP redirect

## One-line definition

DNS maps a name to an **address**; changing the URL bar from one domain to another requires an **HTTP 301**, not a DNS record.

## What it is

A recurring footgun: wanting `example.com` to "point to" `www.other.com` and reaching for DNS to do it. DNS can only answer *"what IP serves this name?"*. The browser's address bar, the TLS SNI, and the HTTP `Host:` header are all set from what the user typed — DNS cannot change any of them. To swap the URL the user sees, some HTTP server at the destination must return a `301 Moved Permanently` with a `Location:` header.

## Why it matters

The wrong instinct — "just add a CNAME" — fails in subtle, bad ways:

- **`@ CNAME other-domain.com` on the apex** — RFC 1912 §2.4 forbids a CNAME on the zone apex (conflicts with SOA/NS). Most registrars reject it outright.
- **CNAME flattening / ALIAS / ANAME** (Cloudflare, Route 53 ALIAS, DNSimple ALIAS) — allowed at the apex, but only resolves to the *IP* of the target. The browser still shows the source domain, sends `Host: source-domain` to that IP, and the destination server either 404s, serves the wrong vhost, or fails TLS (cert won't cover the source name).
- **`A` record pointing to the destination's IP** — same failure mode as flattening, plus the destination's IP can change under you.

Rule of thumb: **"URL bar changes" = HTTP 301, not DNS.** DNS just picks which server answers.

## Implementation patterns

Three practical ways to do a real redirect, cheapest first:

### 1. Registrar URL forwarding

Most registrars (Squarespace, Namecheap, Porkbun, Cloudflare Registrar, former Google Domains) ship a built-in forwarder. Set source apex + `www` → `https://www.destination.com`, mode "permanent (301)", "forward path" on.

**Caveat:** many registrar forwarders don't serve HTTPS cleanly on the *source* name — the browser may warn before the redirect. Fine for a parked/typo domain, not for a brand-primary name.

### 2. Cloudflare Redirect Rules (recommended)

Clean HTTPS end-to-end, free.

1. Add the source zone to Cloudflare; change registrar NS.
2. Create a dummy proxied A record: `@` and `www` → `192.0.2.1`, both orange-clouded. (The IP is irrelevant — Cloudflare intercepts before the origin.)
3. **Rules → Redirect Rules** → *"If hostname equals `source.com` OR `www.source.com`"* → dynamic redirect to:

   ```
   concat("https://www.destination.com", http.request.uri.path)
   ```

   Status **301**, preserve query string.
4. Cloudflare issues a free Universal cert for the source, so `https://source.com/anything` works and 301s path-preserving to `https://www.destination.com/anything`.

### 3. Redirect host you control

nginx `return 301 https://www.destination.com$request_uri;`, S3 static website with redirect rules + CloudFront, Vercel/Netlify `_redirects`, Caddy `redir`. Point DNS at that host. Overkill unless you already run one — but gives you full control (per-path rules, logging, auth-aware redirects).

## Key distinctions

- **DNS record** answers "what IP?" — DNS clients and resolvers never see URLs, paths, query strings, or the user-visible hostname.
- **HTTP redirect** returns `3xx` + `Location:` from an actual web server. Only this changes the URL bar.
- **CNAME flattening** ≠ redirect — it's a server-side DNS trick to let the apex behave like a CNAME. Still returns an IP, still leaves the source hostname intact in the HTTP request.
- **`Host` header** is set by the browser from what the user typed, not by DNS. A server receiving `Host: source.com` has no idea the user "meant" the destination unless configured for it.
- **TLS cert matching** happens against SNI (= source hostname). The destination's cert won't cover the source name unless you add it as a SAN there.

## When this comes up

- Vanity / typo domains redirecting to the canonical site (`philippines-resource.com` → `www.philippines-resources.com`).
- Marketing domains redirecting to product domains.
- Post-rename / post-acquisition domain consolidation.
- Apex-to-`www` canonicalization (same site; still an HTTP 301, not DNS).

## Sources

- RFC 1912 §2.4 — Common DNS Operational and Configuration Errors (CNAME-at-apex prohibition).
- RFC 7231 §6.4.2 — HTTP/1.1 `301 Moved Permanently` semantics.
- Direct conversation 2026-10-09 — `philippines-resource.com` → `www.philippines-resources.com` worked example.
