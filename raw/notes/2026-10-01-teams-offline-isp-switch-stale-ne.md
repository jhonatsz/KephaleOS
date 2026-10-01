---
type: source
status: compiled
source-kind: note
created: 2026-10-01
captured-at: 2026-10-01T20:30
compiled-at: 2026-10-01
compiled-into:
  - "[[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]]"
  - "[[00-dashboard/knowledge-map]]"
---

# Teams offline after ISP switch — Zscaler stale NE claim

## Trigger

After switching ISP on personal Mac, Microsoft Teams showed **"offline"** despite internet working normally. Zscaler **Internet Security** and **Private Access** were both toggled **off** in the GUI; FortiClient GUI was closed.

## Observed evidence

- `nc -zv login.microsoftonline.com 443` → **succeeded**
- `nc -zv teams.microsoft.com 443` → `connectx failed: Can't assign requested address` (`EADDRNOTAVAIL`), returned in **~15 ms** (not a timeout)
- `nc -zv presence.teams.microsoft.com 443` → same `EADDRNOTAVAIL`
- `traceroute 52.123.224.12` → `traceroute: bind: Can't assign requested address`
- `route -n get 52.123.224.12` → `interface: en0, gateway: 192.168.68.1` — **clean**, no CGNAT hijack
- `ifconfig` → six `utun*` interfaces up with **only IPv6 link-local**, no IPv4 (notably `utun0` mtu 1380 — Zscaler Tunnel iface)
- `pgrep ZscalerTunnel` → running (pid 50960)
- `systemextensionsctl list` → `com.fortinet.forticlient.macos.vpn.nwextension [activated enabled]`
- `curl ip.zscaler.com` → still routed through Zscaler enforcement node (service active below the GUI toggle)
- LaunchDaemons loaded: `com.fortinet.forticlient.ztnafw`, `.vpn`, `.servctl2`, `.config`, `.PrivilegedHelper`, `.fssoagent`

## Diagnosis

**Zscaler Tunnel NetworkExtension held a stale route-claim** for Teams signaling IPs (52.123.0.0/16). The ISP switch invalidated the tunnel's local address context (utun4 had no IPv4), but the NE kept the destination-included-routes list live. Kernel socket binds for claimed destinations therefore returned `EADDRNOTAVAIL` **immediately** — not a packet-level block, a socket-layer refusal because no valid source IP could be assigned.

login.microsoftonline.com (40.126.x.x) was NOT in Zscaler's claim list → unaffected throughout.

## Fix

```bash
sudo pkill -HUP ZscalerTunnel
# wait ~5s for respawn
```

Operator also cycled Wi-Fi (`networksetup -setnetworkserviceenabled Wi-Fi off/on`), but the direct evidence of resolution is in the ZscalerTunnel state diff:

| Metric | Before | After |
| --- | --- | --- |
| `ZscalerTunnel` PID | 50960 | 49735 (respawned) |
| `utun4` IPv4 | absent | `100.64.0.1` |
| `nc teams.microsoft.com 443` | `Can't assign requested address` | `succeeded` |

`pkill -HUP ZscalerTunnel` is the surgical fix; Wi-Fi cycle alone may be insufficient if the NE ignores the reachability event.

## Why the Zscaler GUI toggle doesn't fix it

- GUI toggles (Internet Security / Private Access) control proxy/tunnel **engagement**, not NE **registration**.
- The destination-claim list lives in the system-level NEPacketTunnelProvider, loaded by `com.zscaler.tunnel` LaunchDaemon, which the user-mode GUI cannot bootout.
- So toggling off ZIA/ZPA does nothing for a stale route-claim; the claim persists until the daemon itself respawns.

## Why FortiClient was ruled out despite being installed

- `fctservctl2` running, ZTNA FW daemon loaded, NE activated — all consistent with a FortiClient suspect.
- But the actual tunnel state diff proving fix came from **ZscalerTunnel** respawn (new PID + utun4 IPv4 appearing). FortiClient state did not change during the fix window.
- The ZTNA FW remains a latent risk (would exhibit the same symptom shape for destinations in Fortinet's claim list) but was not the active culprit today.

## New failure-mode fingerprint (distinct from prior incidents)

Prior ZPA hijack incidents (2026-09-12, -09-15, -09-22) all presented with `route -n get` returning `utun*` / CGNAT gateway — a **route-table hijack**. Today's incident is a different shape:

- `route -n get` shows **clean** physical interface
- Failure is `EADDRNOTAVAIL` **immediately** on socket bind (not timeout, not RST)
- Trigger is a **network context change** (ISP switch, Wi-Fi reassociate) that the NE doesn't cleanly re-register through

Same culprit family (Zscaler Client Connector), different enforcement layer (NE socket-bind refusal, not FIB divert). Fix family is also different: not `sudo route delete/add`, but `sudo pkill -HUP ZscalerTunnel`.

## Incident count

This is the **4th** Zscaler-caused destination disruption on this machine in ~20 days (prior: FortiGate IPsec, partner SFTP, EC2 CI runner SSH). Teams signaling (52.123/16) is a new 4th destination class, and ISP switch is a newly-identified trigger.

The operator's documented preference (2026-10-01 earlier) is "stick to the route command" — but that preference assumes the route-hijack shape. Today's shape (stale NE) does not respond to the route command. The quick-fix family therefore has to grow: `route` for FIB hijack, `pkill -HUP ZscalerTunnel` for stale NE.
