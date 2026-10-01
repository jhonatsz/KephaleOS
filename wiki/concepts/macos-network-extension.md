---
type: concept
status: active
created: 2026-10-02
updated: 2026-10-02
aliases:
  - "NetworkExtension"
  - "NetworkExtension framework"
  - "NEPacketTunnelProvider"
  - "NEIPv4Settings"
  - "macOS Network Extension"
  - "NE"
tags: [macos, networking, vpn, ztna, concept]
sources:
  - "[[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]]"
  - "[[wiki/runbooks/macos-vpn-ne-triage]]"
confidence: high
---

# macOS NetworkExtension (NE)

## One-line definition

Apple's user-space framework that lets third-party apps register tunnel endpoints *above the kernel route table* — the mechanism that makes `route get` **lie** about where your packets actually go.

## What it is

`NetworkExtension.framework` is Apple's replacement for kernel network extensions (which required kext signing + SIP exceptions). It lets apps register themselves as:

- **Packet-tunnel providers** (`NEPacketTunnelProvider`) — present a `utun*` interface to the kernel, intercept selected destinations
- **App-proxy providers** — intercept socket calls from specific apps
- **Content-filter providers** — observe or block traffic at policy
- **DNS proxy providers** — intercept and rewrite DNS queries

Zscaler Client Connector, FortiClient, Cloudflare WARP, Tailscale, Twingate, Netskope all install themselves as `NEPacketTunnelProvider` system extensions.

The critical structural detail: when a packet-tunnel provider registers, it declares an **`includedRoutes`** list (via `NEIPv4Settings.includedRoutes` / `NEIPv6Settings.includedRoutes`). The kernel consults this list in the socket layer, **before** standard route-table (FIB) lookup. So:

```
Socket connect() →
    NE includedRoutes check (user-space provider) →
        if match → hand to tunnel app
        if no match → FIB lookup (traditional route table) →
            physical interface
```

This two-stage design is why `route -n get <ip>` can show a clean physical interface while the actual connection fails: the NE claimed the destination at the earlier stage, and `route get` only queries the FIB.

## Why it matters

**NE-level route claims survive GUI "off" toggles.** The user-facing VPN switch in Zscaler/FortiClient usually controls *tunnel engagement*, not NE *registration*. If the NE is still active and claiming destinations, those destinations still fail — even when the GUI says the VPN is disconnected.

**NE claims can go stale.** On a network context change (ISP switch, Wi-Fi reassociate, carrier handoff), a well-behaved NE re-registers against the new interface and reassigns a local address on its `utun`. A buggy NE keeps the destination claim but loses the local address → packets to claimed destinations fail instantly with `EADDRNOTAVAIL` because the kernel can't bind a source IP for them. This is **Mode B** in [[wiki/runbooks/macos-vpn-ne-triage]].

Fixing a stale NE requires the daemon to actually restart, not just the GUI to toggle. On Zscaler specifically: `sudo pkill -HUP ZscalerTunnel` triggers a fresh registration.

## Key distinctions

- **Not a kernel extension.** Pre-Catalina, Zscaler/FortiClient shipped kexts. NE is user-space (plus a tiny kernel shim) and lives in the **System Extensions** panel of Settings, not Kernel Extensions.
- **Not a VPN config.** A VPN *configuration* (in `/Library/Preferences/com.apple.networkextension*`) is a declaration; the NE *provider* is the running process. Deleting the config doesn't deactivate the provider.
- **Not controlled by `pf` or `route`.** User-level `route` or `pfctl` changes don't affect NE route claims — the NE sits above them in the connect() path.
- **Not disabled by uninstalling the GUI app.** On macOS, system extensions require `systemextensionsctl deactivate <team-id> <bundle-id>` + user approval; a drag-to-Trash of the `.app` leaves the NE loaded until reboot (and often across reboots).

## Observing NE state

```bash
systemextensionsctl list                            # active system extensions, by team ID
pgrep -lf ZscalerTunnel                             # Zscaler NE provider process
ifconfig | awk '/^utun/{iface=$1} /inet /{print iface, $2}'   # which utun has an IPv4 → active NE tunnels
log show --last 5m --predicate 'process == "ZscalerTunnel"' --style compact
```

The combination of (a) `systemextensionsctl` showing *activated enabled* and (b) `pgrep` returning a PID is the "NE is live" fingerprint. Only (a) without (b) is "configured but not running"; both without `utun` IPv4 is "half-registered — stale claim risk."

## In the vault

NE as a mechanism explains the Mode B failure shape in the ZPA synthesis:

- [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] — Mode B root cause: *"the ZCC NE sits above the FIB in the socket path (via `NEIPv4Settings.includedRoutes`)…"*
- [[wiki/runbooks/macos-vpn-ne-triage]] — Mode B diagnostic (`EADDRNOTAVAIL` in <100 ms with a clean `route get`) and fix (`pkill -HUP ZscalerTunnel`)
- [[raw/notes/2026-10-01-teams-offline-isp-switch-stale-ne]] — the lived incident that revealed the stale-NE failure mode

Before this concept page existed, each ZPA page restated what NE is. Those references can now point here.

## Related concepts

- [[cgnat-100.64.0.0-10|CGNAT (100.64.0.0/10)]] — the address space NE packet-tunnel providers use for their local `utun` IPs

## Sources

- Apple developer docs — `NetworkExtension` framework reference (public API, stable since Catalina)
- See `sources:` frontmatter for the lived-experience pages where NE behavior drove the diagnosis
