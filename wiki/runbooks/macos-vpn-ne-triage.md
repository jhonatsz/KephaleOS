---
type: runbook
status: active
created: 2026-10-02
updated: 2026-10-02
aliases:
  - "Runbook: macOS VPN/NE triage"
  - "Why can't I reach this one host on macOS"
  - "Zscaler Teams offline quick triage"
  - "Zscaler route hijack quick triage"
  - "EADDRNOTAVAIL quick triage"
tags: [runbook, macos, networking, vpn, zscaler, forticlient, ztna, triage]
last-executed: 2026-10-01
sources:
  - "[[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]]"
  - "[[wiki/decisions/2026-10-01-zpa-escalation-deferral]]"
---

# Runbook: macOS VPN/NE triage — one destination won't connect

## When to use

A single destination (or set of destinations) can't be reached from your
macOS workstation while **the rest of the internet works fine**, and
you have any of these agents installed: **Zscaler Client Connector**,
FortiClient, Cloudflare WARP, Tailscale, Netskope, Twingate, GlobalProtect.

Typical symptoms as users describe them:

- *"MS Teams went offline but my internet works"*
- *"My VPN won't connect, but I can ping its gateway"*
- *"SSH to our EC2 bastion hangs; same key works from my phone's hotspot"*
- *"This one SFTP endpoint keeps saying connection refused"*

If **every** destination fails: this isn't the right runbook — check Wi-Fi,
DNS (`scutil --dns`), and whether a tunnel is forcing full-tunnel. If
**only web** fails but TCP works: suspect ZIA proxy policy, not this.

## Prerequisites

- Terminal
- Target destination's IP and port (or a hostname that resolves)
- Known-good control destination (e.g. `1.1.1.1:443`)
- Optionally `sudo` for the Mode A fix

## 10-second diagnosis

```bash
TARGET=teams.microsoft.com                   # or your destination
PORT=443
IP=$(dig +short $TARGET | head -1)

nc -zv -G 3 $IP $PORT                        # measure failure SHAPE
route -n get $IP                             # measure routing CLAIM
```

Read the two outputs together:

| `nc` result | `route get` interface | Mode | Section |
| --- | --- | --- | --- |
| `succeeded` | *any* | ✅ not this runbook | — |
| **Immediate** `Can't assign requested address` (<100 ms) | physical (`en0`/`en1`) | **B — stale NE** | §Mode B |
| **TCP RST** / "Connection refused" (<1 s) | `utun*` or `100.64.0.*` | **A1 — FIB hijack, no allow rule** | §Mode A |
| **Silent timeout** (>5 s) | `utun*` or `100.64.0.*` | **A2 — FIB hijack, forwarded into dead tunnel** | §Mode A |
| Silent timeout | physical | ⚠️ not this pattern | §Not this |

The two measurements together are the fingerprint. `nc` alone can't tell you
*why*; `route get` alone can't tell you *whether* traffic is actually dying
at the claimed interface. Always run both.

## Mode A — FIB route hijack (fix in 30 s)

**Fingerprint recap.** `route get <ip>` shows `interface: utun*` or
`gateway: 100.64.0.*` ([[wiki/concepts/cgnat-100.64.0.0-10|CGNAT space]]
— a client-side tunnel's local address). Classic Zscaler ZPA / Cloudflare
WARP / Tailscale pattern: the agent installed a per-host route stealing
traffic into its own tunnel.

**Fix:**

```bash
LAN_GW=$(netstat -nr -f inet | awk '/^default/ {print $2; exit}')
sudo route -n delete $IP
sudo route -n add -host $IP -gateway $LAN_GW
route -n get $IP                               # verify: interface=en0 (or your physical NIC)
nc -zv $IP $PORT                               # expect: succeeded
```

**Caveats.**

- Dies on reboot, Wi-Fi change, or the offending agent's next policy sync.
- Zscaler with strict enforcement can block user route changes; if `route add`
  silently reverts, the local fix won't stick → ask IT for an App Segment
  bypass instead (see §Durable fix).

## Mode B — stale [[wiki/concepts/macos-network-extension|NetworkExtension]] (fix in 10 s)

**Fingerprint recap.** `route get <ip>` looks clean (physical interface,
LAN gateway) **but** `nc` returns `Can't assign requested address` in
under 100 ms. The ZCC NetworkExtension still holds the destination
route-claim but lost its tunnel's IPv4 source — kernel can't bind. See
the [[wiki/concepts/macos-network-extension|NE concept page]] for the
two-stage connect() diagram that explains why `route get` reports clean
while the connection still fails.

**Why it happens.** Network context changed (ISP switch, Wi-Fi reassociate,
carrier handoff, laptop wake-from-sleep across networks) and the NE's
re-registration half-failed.

**Fix:**

```bash
sudo pkill -HUP ZscalerTunnel
sleep 5
ifconfig | awk '/^utun/{iface=$1} /inet 100\./{print iface, $2}'  # expect utun* with 100.64.0.1 to reappear
nc -zv $IP $PORT                                                    # expect: succeeded
```

If `ZscalerTunnel` doesn't exist on your box, substitute the equivalent:

- FortiClient ZTNA FW: `sudo launchctl kickstart -k system/com.fortinet.forticlient.ztnafw`
- Cloudflare WARP: `warp-cli disconnect && warp-cli connect`

**Caveats.**

- Toggling the Zscaler GUI (Internet Security / Private Access to Off) does
  **not** clear the stale claim — the NE lives below the GUI layer. The
  daemon has to actually restart.
- If `pkill -HUP` doesn't respawn `ZscalerTunnel` (check with `pgrep ZscalerTunnel`),
  cycle Wi-Fi: `networksetup -setnetworkserviceenabled Wi-Fi off; sleep 3; networksetup -setnetworkserviceenabled Wi-Fi on`.
- Reboot always works. Use when nothing else does.

## Not this runbook

- `route get` physical + silent timeout → genuine network reachability
  problem; test with a known-good host, check Wi-Fi/DHCP, check the
  destination's own status.
- All destinations fail → full-tunnel VPN engaged or Wi-Fi/DNS broken.
- Only HTTPS fails but TCP works → ZIA proxy TLS inspection policy; not
  solvable from the workstation.
- Login works but one SaaS fails → confirm with the SaaS admin that your
  user has access; this runbook is for *routing*, not authorization.

## After the fix

- **Note the mode and destination.** If you're seeing a new destination
  class or a new mode, it's incident evidence — append a line to the
  "Recurrence" table on [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]].
- **Check revisit triggers.** [[wiki/decisions/2026-10-01-zpa-escalation-deferral|The deferral decision]]
  carries five revisit conditions (recurrence rate ≥1/week sustained,
  non-macOS-routable destination, user-facing miss, work-laptop
  migration slip past 2026-12-31, EMS scope broadening). If any fire,
  escalate — don't just reach for this runbook again.

## Durable fix (not operator-side)

The quick fixes in this runbook are **local workarounds**. The structural
fixes are not on your side of the fence:

- **For Zscaler ZPA:** IT adds the destination to the Application Bypass
  list, or removes it from the App Segments. ~30 seconds in the admin
  console; survives reboot and policy sync.
- **For Cloudflare WARP:** admin adds the IP/CIDR to the Split Tunnel
  Exclude list.
- **For FortiClient ZTNA FW:** admin edits the policy pushed from EMS.

See [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] §Fix
for the full policy-side playbook.

## Related

- [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] — deep page: full root-cause explanation, incident history, two-enforcement-layer taxonomy, policy-side durable fixes
- [[wiki/decisions/2026-10-01-zpa-escalation-deferral]] — why "run this runbook again" is the current chosen path over IT escalation, and what would flip the ranking

## Sources

See the deep solution page and the deferral decision for incident-level
provenance (four lived cases spanning both modes, 2026-09-12 → 2026-10-01).
