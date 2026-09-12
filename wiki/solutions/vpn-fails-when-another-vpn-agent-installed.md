---
type: solution
status: active
created: 2026-09-12
updated: 2026-09-12
tags: [vpn, networking, macos, zscaler, forticlient, ztna, ipsec, routing]
sources:
  - "[[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]]"
confidence: high
---

# Solution: VPN client fails to connect while another VPN/ZTNA agent is installed

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

Source packets appear to originate from a **CGNAT** address (100.64.0.0/10)
rather than the physical interface's address.

## Root cause

A second VPN/ZTNA agent (**Zscaler Client Connector**, Cloudflare WARP,
Tailscale, Twingate, Netskope, ProtonVPN's split-tunnel proxy, etc.) has
installed a **per-host route** that steals traffic to the target VPN
gateway into its own tunnel. IKE / control-plane packets never reach the
real gateway, so the outer VPN never completes negotiation.

Specifically for Zscaler ZPA: if the ZPA App Segment list includes (or
overlaps) the outer VPN's gateway IP, ZCC's packet-tunnel provider
programs a host route to `100.64.0.1` (Z-Tunnel) for that destination,
overriding the default route.

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

- [[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]] — the incident this was compiled from

## Sources

- Live troubleshooting session, 2026-09-12 (see raw note)
