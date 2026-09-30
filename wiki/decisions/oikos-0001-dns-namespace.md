---
type: decision
status: active
created: 2026-09-18
updated: 2026-09-18
aliases: [oikos-dns-namespace, oikos-internal-zone]
tags: [oikos, adr, dns, namespace, rfc-8375]
confidence: high
---

# ADR oikos-0001 — DNS namespace for the Oikos internal zone

## Status

**Accepted** — 2026-09-18

## Context

The Oikos internal DNS zone needs a canonical name. Two candidates were in play when the T14 install began:

- `oikos.home.arpa` — the design docs' choice from 2026-09-18
- `homelab.arpa` — what the T14 was momentarily installed with

**RFC 8375** ("Special-Use Domain 'home.arpa.'") reserves `home.arpa` for use as the default domain name for residential home networks. Subdomains of `home.arpa` (like `oikos.home.arpa`) are the intended pattern for anyone who wants a labelled internal namespace.

`.arpa` itself is the ARPA infrastructure zone administered by IANA, used for reverse DNS (`in-addr.arpa`, `ip6.arpa`) and other RFC-assigned special-purpose subdomains. Using an arbitrary label directly under `.arpa` (e.g. `homelab.arpa`) is not compliant with the reserved-use rules — although it will not collide with anything today, it might if IANA delegates that label in the future.

## Decision

The Oikos internal DNS zone is **`oikos.home.arpa`**. All hosts, services, certificates, DHCP option 15 values, AD DNS suffixes, and Prometheus targets use this suffix.

Hostname pattern: `<role>-<hw-tag>-<index>.oikos.home.arpa`.

## Alternatives considered

- **`homelab.arpa`** — rejected. Non-compliant with RFC 8375 (bare `.arpa` subdomains are IANA-managed). Would require rewriting every doc.
- **`home.arpa` (unlabelled)** — rejected. Compliant but loses the "Oikos" identity in every FQDN and pre-commits the whole reserved zone to this one project. If a future household project ever wants its own internal zone, they'd have to co-tenant.

## Consequences

- One-time hostname rename on `pve-t14-01` (in-flight install; ~60 seconds — see "Applying" below).
- All future Oikos hosts named per the pattern above without exception.
- Reverse DNS still uses standard `in-addr.arpa` delegation for `172.27.0.0/16`.
- The internal zone is served by internal DNS only (Phase 4). It never leaves the household network. External DNS is unaffected.
- Any subsequent household project (unrelated to Oikos) can pick its own `<name>.home.arpa` subdomain without collision.

## Applying to the T14 (in flight)

On `pve-t14-01`, as root:

```bash
hostnamectl set-hostname pve-t14-01.oikos.home.arpa
```

Then edit `/etc/hosts` — the `127.0.1.1` line must match the new FQDN and short name:

```
127.0.0.1       localhost
127.0.1.1       pve-t14-01.oikos.home.arpa pve-t14-01
```

Verify:

```bash
hostname -f    # → pve-t14-01.oikos.home.arpa
hostname -s    # → pve-t14-01
```

If Proxmox has already generated certs against the old name, regenerate:

```bash
pvecm updatecerts --force
systemctl restart pveproxy pvedaemon
```

Nothing else depends on the hostname yet — safe to change during Phase 0.

## Related

- [[../projects/oikos]]
- `~/kephaleos/work/projects/oikos/roadmap/phase-00-foundation.md`
- `~/kephaleos/work/projects/oikos/nodes/pve-t14-01.md`
- RFC 8375 — Special-Use Domain 'home.arpa.'
