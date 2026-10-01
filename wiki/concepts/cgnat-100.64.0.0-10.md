---
type: concept
status: active
created: 2026-10-02
updated: 2026-10-02
aliases:
  - "CGNAT"
  - "Carrier-Grade NAT"
  - "100.64.0.0/10"
  - "RFC 6598"
  - "Shared Address Space"
tags: [networking, ipv4, nat, vpn, ztna, concept]
sources:
  - "[[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]]"
  - "[[wiki/runbooks/macos-vpn-ne-triage]]"
confidence: high
---

# CGNAT (100.64.0.0/10)

## One-line definition

The IPv4 range `100.64.0.0/10` is **not routable on the public internet** — it's reserved for carrier-grade NAT and client-side tunnels; seeing it on your local machine is a fingerprint, not a bug.

## What it is

`100.64.0.0/10` (addresses `100.64.0.0` through `100.127.255.255`) is **RFC 6598 Shared Address Space** — IANA-allocated, globally non-routable, meant for use *inside* a service provider's NAT boundary where RFC 1918 ranges (`10/8`, `172.16/12`, `192.168/16`) would collide with customer-side networks.

Two populations use it:

- **ISPs doing carrier-grade NAT** (mobile carriers, apartment-building ISPs, satellite providers) — the "100.*" you'd see as your public IP on cell data
- **Client-side tunnel software** that needs a stable local address space without colliding with the customer's LAN — Zscaler Z-Tunnel (`100.64.0.1` on `utun*`), Cloudflare WARP, Tailscale (`100.64.0.0/10` fully, by convention), Twingate, Netskope

Both populations exist because `100.64.0.0/10` is the only officially-reserved space that *reliably* won't collide with a corporate LAN or home router subnet. RFC 1918 can't guarantee that; `100.64/10` can.

## Why it matters

**On macOS/Linux workstations, `100.64.0.0/10` showing up in `ifconfig` on a `utun*` interface is near-pathognomonic for a client-side ZTNA/VPN tunnel.** The operator is almost never behind a carrier CGNAT on a workstation — if they were, it'd be the WAN side, not a tunnel interface.

Operational fingerprinting:

```bash
route -n get <destination-ip> | grep gateway       # if "100.64.0.*" → client-side tunnel claims this destination
ifconfig | awk '/^utun/{iface=$1} /inet 100\./{print iface, $2}'   # which utun has a 100.64 address = the active tunnel
```

See [[wiki/runbooks/macos-vpn-ne-triage]] §Mode A for the fix pattern when the fingerprint matches a hijack.

## Key distinctions

- **Not RFC 1918.** `10/8`, `172.16/12`, and `192.168/16` are private but *routable within a LAN*. CGNAT is a tighter category — specifically reserved for the NAT-boundary layer where private ranges can't be used because of collision risk.
- **Not routable on the public internet.** Backbone routers drop `100.64/10` by policy; you can't `ping` it from the internet even if the host were up.
- **Not loopback or link-local.** `127/8` is loopback, `169.254/16` is link-local (DHCP-failure fallback). `100.64/10` is intentional NAT infrastructure.
- **Not a sign of malware or compromise** on its own — but on a workstation that doesn't run any ZTNA client and isn't on cell data, a `100.64/10` route is unexplained and worth investigating.

## In the vault

`100.64.0.0/10` as the ZCC fingerprint appears across the Zscaler hijack synthesis:

- [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] — Mode A (FIB route hijack) → `route -n get` shows CGNAT next-hop
- [[wiki/runbooks/macos-vpn-ne-triage]] — Mode A diagnostic criterion
- [[wiki/decisions/2026-10-01-zpa-escalation-deferral]] — context (`route -n get` fingerprint drove the deferral rationale)

Before this concept page existed, each page restated what `100.64.0.0/10` means. Those references can now point here.

## Related concepts

- [[macos-network-extension]] — the macOS mechanism Zscaler/FortiClient use to install `utun*` + CGNAT routes

## Sources

- RFC 6598 — IANA-Reserved IPv4 Prefix for Shared Address Space (2012)
- See `sources:` frontmatter for the lived-experience pages where this concept recurred
