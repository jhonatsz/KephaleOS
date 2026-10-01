---
type: solution
status: active
created: 2026-09-12
updated: 2026-10-01
aliases:
  - "Solution: VPN fails when another VPN/ZTNA agent installed"
  - "VPN-over-VPN routing conflict"
  - "Zscaler + FortiClient routing hijack"
  - "Zscaler ZPA route hijack"
  - "ZPA hijacking arbitrary destination"
  - "Zscaler stale NetworkExtension claim"
  - "EADDRNOTAVAIL on Microsoft 365 after ISP switch"
  - "MS Teams offline despite internet working"
tags: [vpn, networking, macos, zscaler, forticlient, ztna, ipsec, routing, sftp, ssh, aws, teams, microsoft-365]
sources:
  - "[[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]]"
  - "[[raw/notes/2026-09-15-zpa-hijack-partner-sftp]]"
  - "[[raw/notes/2026-09-22-zpa-hijack-ec2-runner-ssh]]"
  - "[[raw/notes/2026-10-01-0456-zpa-route-command]]"
  - "[[raw/notes/2026-10-01-teams-offline-isp-switch-stale-ne]]"
confidence: high
---

# Solution: VPN client fails to connect while another VPN/ZTNA agent is installed

## Quick fix (mid-incident)

**First decide which failure mode you're in** — the two modes have different fixes:

```bash
# Diagnose which mode:
route -n get <destination-ip>
nc -zv <destination-ip> 443        # time how fast it fails
```

| Fingerprint | Mode | Quick fix |
| --- | --- | --- |
| `route` → `utun*` / `100.64.0.*` gateway; connect **times out** or returns **TCP RST** | **Mode A — FIB route hijack** | `sudo route delete/add` to LAN gateway (below) |
| `route` → clean (`en0`, LAN gateway); connect fails with `Can't assign requested address` (`EADDRNOTAVAIL`) **in <100 ms** | **Mode B — stale NE claim** | `sudo pkill -HUP ZscalerTunnel` |

### Mode A — route-table hijack

```bash
sudo route -n delete <gateway-ip>
sudo route -n add -host <gateway-ip> -gateway <lan-gateway>  # LAN gateway, e.g. 192.168.68.1
route -n get <gateway-ip>                                    # verify: interface=en0
# → then Connect in the VPN client / retry the destination
```

### Mode B — stale NetworkExtension (after ISP / Wi-Fi switch)

```bash
sudo pkill -HUP ZscalerTunnel
sleep 5
ifconfig | grep -A1 utun | grep "inet "    # expect utun4 to re-appear with 100.64.0.1
nc -zv <destination-ip> 443                # expect "succeeded"
```

Both quick fixes are **temporary**. Mode A dies on reboot / Wi-Fi change /
offending agent's next policy sync. Mode B dies on the next ISP switch
when the NE re-enters a stale state. Read §Fix for the durable path
(admin-side bypass list / policy scope review).

**Third-occurrence rule:** if this pattern hits a third destination on
the same machine, stop filing per-host bypass tickets and ask IT for a
ZPA policy scope review instead.

## Symptom

A VPN client (FortiClient IPsec, Cisco AnyConnect, OpenVPN, WireGuard,
Pulse Secure, GlobalProtect, etc.) fails during connection setup even
though:

- The internet works
- DNS resolves the gateway hostname
- The gateway is reachable by `ping` / `nc -uz`
- The app-level error is vague or misleading (e.g., FortiClient logging
  *"failed to connect to update server"*, or Phase-1 IKE timing out with
  no reply)

Exact strings seen in the wild:

```
phase1 negotiation failed due to time up.
```

Or, for TCP destinations that aren't a VPN gateway at all (arbitrary
services reached via SSH/HTTPS/SFTP):

```
nc: connectx to <host> port <port> (tcp) failed: Connection refused
ssh: connect to host <host> port <port>: Connection refused
```

Source packets appear to originate from a **CGNAT** address (100.64.0.0/10)
rather than the physical interface's address.

### Three symptom shapes — two distinct enforcement layers

Three failure modes observed on the same machine for the same culprit
family (Zscaler Client Connector):

| Symptom | Enforcement layer | What it means |
| --- | --- | --- |
| Silent **timeout** (UDP: IKE Phase-1; TCP: hang) | FIB (route table) | Destination is inside ZCC's tunnel scope; ZCC forwards into Z-Tunnel, other side never answers |
| Fast **TCP RST** (connection refused in ~1 s) | FIB (route table) | Destination route is hijacked but ZCC has no policy rule allowing it; ZCC synthesizes RST |
| **`EADDRNOTAVAIL`** / "Can't assign requested address" in **<100 ms** | Socket layer (NetworkExtension) | ZCC's NEPacketTunnelProvider still holds the destination route-claim but has no valid IPv4 source on its utun (common after ISP/Wi-Fi switch); kernel refuses the socket bind |

**The first two share a fingerprint** — `route -n get <destination>` returns
`utun*` / CGNAT (`100.64.0.0/10`) gateway. Fix: route override to LAN
gateway (Mode A).

**The third is distinct.** `route -n get` returns a clean physical
interface, so the FIB looks correct. But the ZCC NE sits *above* the FIB
in the socket path (via `NEIPv4Settings.includedRoutes`), still claiming
the destination. With no local address on the tunnel, binds fail
instantly. Fix: restart the daemon so the NE re-registers (Mode B).

These distinctions help predict the right policy ask:

- Mode A/B pointing at *external* destinations (partner infra, public SaaS)
  → IT bypass ticket, or scope-review escalation after 3+ incidents.
- Mode A/B pointing at *employer* destinations → sometimes it's a missing
  **App Segment** entry rather than needing a bypass; ask IT which.

## Root cause

A second VPN/ZTNA agent (**Zscaler Client Connector**, Cloudflare WARP,
Tailscale, Twingate, Netskope, ProtonVPN's split-tunnel proxy, etc.)
holds the traffic for the target destination at one of two layers.

**Mode A — FIB route hijack (route-table interception).** The ZTNA agent
has installed a **per-host route** that steals traffic to the target IP
into its own tunnel. For Zscaler ZPA: if the ZPA App Segment list
includes (or overlaps) the destination's IP, ZCC's packet-tunnel
provider programs a host route to `100.64.0.1` (Z-Tunnel), overriding
the default route. Packets reach the tunnel but never the real
destination (or get RST'd if ZCC has no matching allow rule).

**Mode B — stale NetworkExtension socket-layer claim.** macOS routes
destinations into a NetworkExtension via `NEIPv4Settings.includedRoutes`,
which sits **above the FIB** in the socket path. On a network context
change (ISP switch, Wi-Fi reassociate, carrier handoff), the tunnel
should re-register with the new interface's addresses. The known ZCC
bug: the NE **keeps the destination claim** but **loses its own IPv4
assignment** on utun — the kernel then can't bind a source address for
sockets to claimed destinations, returning `EADDRNOTAVAIL` immediately.
The FIB looks clean because the NE's claim doesn't live in it. Only
destinations in the NE claim list are affected; everything else goes
direct and works. GUI toggles for ZIA/ZPA don't clear the stale claim
because they operate above the system LaunchDaemon layer; only a daemon
restart does.

## Fix

### 1. Diagnose in 30 seconds

```bash
# Is the "network problem" actually a routing hijack?
route -n get <vpn-gateway-ip>
```

If `interface:` is anything other than the expected physical interface
(usually `en0` on macOS, `eth0`/`wlan0` on Linux) — or the `gateway:` is in
the 100.64.0.0/10 CGNAT range — a client-side tunnel is intercepting.

Cross-check the gateway is otherwise reachable:

```bash
ping -c 3 <vpn-gateway-ip>
nc -u -v -z -w 3 <vpn-gateway-ip> 500   # IKE
nc -u -v -z -w 3 <vpn-gateway-ip> 4500  # IPsec NAT-T
nc    -v -z -w 3 <vpn-gateway-ip> 443   # SSL-VPN / TCP fallback
```

Identify which tunnel is hijacking:

```bash
ifconfig | grep -E "utun|tun" -A2 | grep "inet "     # macOS
ip -o addr show | grep -E "utun|tun|tailscale"        # Linux
ps -Ao pid,user,comm | grep -iE "zscaler|warp|tailscale|netskope|twingate|proton"
systemextensionsctl list                              # macOS network extensions
```

### 2. Durable fix — coordinate with the offending agent's admin

- **Zscaler ZPA:** IT/admin adds the VPN gateway IP to the **Application
  Bypass** list under *App Profile → Forwarding Profile*, or removes it
  from *ZPA → Application Segments*. ~30-second change in ZCC admin console.
- **Cloudflare WARP:** Split Tunnel → add the gateway IP/CIDR to the
  Exclude list.
- **Tailscale:** exit-node / subnet-router config causing the collision;
  adjust `--advertise-routes` or ACLs.
- **Netskope / others:** equivalent bypass configuration in the client
  policy.

This is the only fix that survives reboots, Wi-Fi changes, and policy
refreshes.

### 3. Temporary local override (fragile)

If IT is slow and you must connect now:

```bash
sudo route -n delete <vpn-gateway-ip>
sudo route -n add -host <vpn-gateway-ip> -gateway <lan-gateway-ip>
route -n get <vpn-gateway-ip>       # verify: interface is your physical NIC
```

Then immediately initiate the VPN connection. Caveats:

- Dies on reboot, network change, or the offending agent's next policy sync
- Some clients (Zscaler with strict enforcement) block user route changes
  entirely — in which case only the admin bypass works
- Pausing the offending agent is often disabled by admin policy

## Verification

- `route -n get <vpn-gateway-ip>` returns the physical interface, not a
  `utun*` device
- Outer VPN completes negotiation (FortiClient status goes to 100%; IPsec
  Phase 1 and Phase 2 succeed in the log)
- `traceroute <vpn-gateway-ip>` first hop is the LAN gateway, not the
  CGNAT tunnel endpoint

## Why this was hard

- **App-level errors mislead.** FortiClient's most visible log line named
  the wrong subsystem ("update server") — the actual Phase-1 timeout was
  buried in `ipsec.log`. Always read the transport/protocol log directly
  before believing the UI's summary error.
- **Reachability tests pass.** `ping` and `nc` may go through the
  hijacking tunnel if it happens to forward that traffic transparently,
  making the network look "fine" while the real problem is routing scope.
  Only `route -n get <destination>` reveals the interception.
- **CGNAT range 100.64.0.0/10 is a fingerprint.** If it appears in
  `ifconfig` on a machine that isn't behind a carrier CGNAT, it's a
  client-side tunnel: Zscaler ZPA (`100.64.0.1`), Cloudflare WARP,
  Tailscale, Twingate, Netskope all live there.

## Lesson

- **Trace the protocol layer, not the app layer.** The most useful log
  file is rarely the one the UI links to.
- **Check the routing table before assuming network failure.** One
  command (`route -n get`) distinguishes "gateway down" from "another
  process stole this route."
- **Client-side VPN agents fight each other by default.** Any deployment
  that mixes ZTNA (Zscaler, Netskope) with a legacy IPsec/SSL-VPN client
  needs an explicit bypass list on the ZTNA side. This is a
  configuration-hygiene issue, not a bug.

## Related

- [[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]] — incident #1 (FortiClient IPsec, FIB hijack, timeout shape)
- [[raw/notes/2026-09-15-zpa-hijack-partner-sftp]] — incident #2 (arbitrary TCP SFTP, FIB hijack, RST shape)
- [[raw/notes/2026-09-22-zpa-hijack-ec2-runner-ssh]] — incident #3 (AWS EC2 CI runner, SSH 22, FIB hijack, silent timeout shape)
- [[raw/notes/2026-10-01-teams-offline-isp-switch-stale-ne]] — incident #4 (MS Teams signaling, **stale NE** shape — new mode)

## Recurring-problem note

As of 2026-10-01 this pattern has bitten **four times in ~20 days** on
the same machine, against four unrelated destination classes and spanning
**two distinct enforcement layers**:

| Date | Destination | Owner | Mode | Symptom |
| --- | --- | --- | --- | --- |
| 2026-09-12 | Corp FortiGate (IPsec) | Employer | A (FIB) | UDP IKE timeout |
| 2026-09-15 | Partner SFTP (TCP 2233) | Third party | A (FIB) | Fast TCP RST |
| 2026-09-22 | Employer AWS EC2 CI runner (SSH 22) | Employer | A (FIB) | Silent TCP + ICMP timeout |
| 2026-10-01 | MS Teams signaling (52.123/16) | Microsoft public SaaS | **B (stale NE)** | `EADDRNOTAVAIL` after ISP switch; cleared by `pkill -HUP ZscalerTunnel` |

**Third-occurrence rule triggered.** Stop filing per-host bypass tickets.
The right ask is a **ZPA policy scope review** with IT: ZCC's hijack
scope (currently covering large public-internet ranges — one observed
route was `32.0.0.0/3`) is far broader than the ZPA App Segments
justify. Either narrow the scope, or maintain an explicit bypass list
of the external endpoints this user actually reaches for legitimate
work. Continuing per-host tickets is now a signal that the ZPA policy
itself is misconfigured, not that individual destinations are
exceptions.

**When the target is employer-owned infrastructure** (as in the 3rd
incident), be aware two failure modes can stack: the ZPA hijack AND
a security-group rule that only allow-lists corp NAT. Resolve the
hijack first, then confirm the SG accepts the resulting egress IP —
or connect FortiClient IPsec so the egress goes through corp NAT
directly, avoiding both problems in one step.

### In practice (2026-10-01)

The escalation ask above is what the third-occurrence rule *recommends*.
What the operator *actually does* is different: keep using the local
`route` command each time the hijack fires, and skip the IT
conversation. Operator's own note (2026-10-01, morning):

> "for ZPA i just stick to route command to fix it"

Two things this tells future-me:

1. **The friction ranking is real.** A 30-second `route` command
   beats scheduling a policy-scope review conversation with IT, even
   at three incidents in ten days. Whenever this page's "third-
   occurrence rule" fires again, expect the operator to reach for the
   local fix first. The recommendation above is technically correct
   and stays on the page — but it's rarely the path taken.
2. **The incident count will keep growing.** Because the local fix
   doesn't change the ZPA policy, the next unrelated destination that
   trips the hijack will fire again and land here as incident #4, #5,
   etc. If/when the count reaches a threshold where the friction
   ranking flips (e.g. the workaround is no longer 30 seconds because
   the target isn't macOS-routable, or the destination is a shared
   resource where the operator can't unilaterally patch the local
   route), *that* is when the escalation actually happens. Track this
   in the table above.

**Update (2026-10-01, evening) — incident #4 landed, and it needed a
different quick-fix than the operator's documented preference.** MS Teams
went offline after an ISP switch; `route` command was *not* the fix
because the FIB was already clean. The failure mode was the stale-NE
one (Mode B), which only responds to `pkill -HUP ZscalerTunnel`. This
means the operator's "I just stick to the route command" heuristic is
necessary-but-not-sufficient. The expanded quick-fix family is now:

- Mode A (FIB hijack, `utun*` in `route get`) → `route delete/add`
- Mode B (stale NE, `EADDRNOTAVAIL` with clean route) → `pkill -HUP ZscalerTunnel`

Diagnosis takes ~10 seconds (`route -n get <ip>` + `nc -zv <ip> <port>`)
and determines which. The friction-ranking argument still stands — both
quick fixes are faster than an IT ticket — but the Mode B evidence
weakens the "local fixes are always enough" case: this new mode has a
different trigger surface (ISP switch) that will keep firing on travel
days and tethering, so the recurrence rate may rise.

Source: [[raw/notes/2026-10-01-0456-zpa-route-command]] (morning
preference statement) and [[raw/notes/2026-10-01-teams-offline-isp-switch-stale-ne]]
(evening incident #4).

## Sources

- Live troubleshooting session, 2026-09-12 (see raw note)
- Live troubleshooting session, 2026-09-15 (see raw note)
- Live troubleshooting session, 2026-09-22 (see raw note)
- Operator practice note, 2026-10-01 morning — chooses local `route` fix
  over IT escalation (see raw note)
- Live troubleshooting session, 2026-10-01 evening — Teams offline after
  ISP switch, new Mode B stale-NE failure identified and fixed via
  `pkill -HUP ZscalerTunnel` (see raw note)
