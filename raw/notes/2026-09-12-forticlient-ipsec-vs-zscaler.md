---
type: source
status: stable
created: 2026-09-12
tags: [vpn, zscaler, forticlient, macos, routing, incident]
---

# FortiClient IPsec Phase-1 timeout while Zscaler installed (macOS)

Raw evidence from a live troubleshooting session. Gateway IP redacted
because this vault is committed to git; keep the pattern, not the target.

## Environment

- macOS (Darwin 25.5.0, arch Apple Silicon)
- FortiClient 7.4.3 (network extension `com.fortinet.forticlient.macos.vpn.nwextension`)
- Zscaler Client Connector installed with **admin-locked uninstall passcode** (employer-deployed on personally-owned hardware)
- Home LAN: `en0 = 192.168.68.55`, default gateway `192.168.68.1`, egress a residential ISP
- Target: employer FortiGate at `<gateway-ip>` (public IP, redacted)

## Initial (misleading) symptom the user reported

> "FortiClient logs say failed to connect to the update server"

That log line is about **FortiGuard update endpoints**, not the VPN tunnel. It
is a red herring and unrelated to Phase-1 failures.

## Real error, buried in the correct log

`/Library/Application Support/Fortinet/FortiClient/Logs/ipsec.log`:

```
2026-09-12 20:31:58: INFO: accept a request to establish IKE-SA: <gateway-ip>
2026-09-12 20:31:58: INFO: initiate new phase 1 negotiation: 100.64.0.1[4500]<=><gateway-ip>[4500]
2026-09-12 20:31:58: INFO: begin Aggressive mode.
2026-09-12 20:33:03: ERROR: phase1 negotiation failed due to time up. 3eeb6dab699ed262:0000000000000000
```

Repeated across every retry. Key detail: **source address `100.64.0.1[4500]`** —
that is CGNAT range (100.64.0.0/10), not the LAN address (192.168.68.55).

## Diagnostic that revealed the cause

Reachability from the physical interface was fine — proving gateway was up
and network path was clean:

```bash
$ ping -c 3 <gateway-ip>            # 3/3, ~94ms
$ nc -u -v -z -w 3 <gateway-ip> 500  # succeeded
$ nc -u -v -z -w 3 <gateway-ip> 4500 # succeeded
$ nc -v -z -w 3 <gateway-ip> 443     # succeeded
```

But the routing table told a different story:

```bash
$ route -n get <gateway-ip>
   route to: <gateway-ip>
    gateway: 100.64.0.1
  interface: utun4          # ← wrong: should be en0
      flags: <UP,GATEWAY,HOST,DONE,WASCLONED,IFSCOPE,IFREF>

$ route -n get default
    gateway: 192.168.68.1
  interface: en0            # ← correct default route
```

A **per-host route override** was steering the FortiGate IP into `utun4`.
`ifconfig utun4` showed `inet 100.64.0.1 --> 100.64.0.1 netmask 0xffff0000`.

## Identifying the culprit

```bash
$ ps -Ao pid,user,comm | grep -iE "vpn|tunnel|zscaler|warp|tailscale|…"
  555 root  /Library/Frameworks/OVPNHelper.framework/…/ovpnhelper
56680 root  /Applications/Zscaler/Zscaler.app/Contents/PlugIns/ZscalerService
57640 root  /Applications/Zscaler/Zscaler.app/Contents/PlugIns/ZscalerTunnel
…
$ systemextensionsctl list
… com.fortinet.forticlient.macos.vpn.nwextension     [activated enabled]
… ch.protonvpn.mac.WireGuard-Extension               [activated waiting for user]
… ch.protonvpn.mac.Transparent-Proxy                 [activated waiting for user]
```

`ZscalerTunnel` owned utun4. Zscaler's Z-Tunnel had the FortiGate IP in its
app-segment list (ZPA private-app routing), so the packet-tunnel provider
installed a host route claiming that destination.

## Fix applied (temporary)

```bash
sudo route -n delete <gateway-ip>
sudo route -n add -host <gateway-ip> -gateway 192.168.68.1
route -n get <gateway-ip>   # verify interface: en0
# then hit Connect in FortiClient — Phase 1 completed
```

Confirmed working. Caveats:
- Dies on reboot, Wi-Fi change, or Zscaler policy refresh
- Some Zscaler deployments block user route changes; then only the ZCC-admin bypass works

## Fix required (durable)

Employer IT to add the FortiGate IP to the Zscaler bypass list. Options:
- ZCC admin console → App Profile → Forwarding Profile → Application Bypass, or
- ZPA → Application Segments → exclude/remove that IP, or
- PAC Bypass for ZIA if using PAC forwarding

## Adjacent observation (not resolved)

Proton VPN's WireGuard network extension is also present but in
`activated waiting for user` state — inactive during this incident but a
future collision candidate if enabled.

## Meta-notes (used for wiki synthesis)

- App-level error text ("failed to connect to update server") pointed at
  the wrong subsystem. Cost ~10 minutes of misdirection.
- Reading `ipsec.log` directly + running `route -n get <destination>` were
  the two moves that resolved it. Both are generic to any VPN failure.
- CGNAT range 100.64.0.0/10 appearing in `ifconfig` on a machine that
  isn't behind CGNAT is a strong fingerprint of a hidden client-side tunnel
  (Zscaler ZPA, Tailscale, Cloudflare WARP, Twingate all use it).
