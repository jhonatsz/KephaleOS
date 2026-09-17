---
type: source
status: stable
created: 2026-09-15
tags: [vpn, zscaler, macos, routing, sftp, incident, ztna]
---

# Second confirmed Zscaler ZPA route hijack — partner SFTP endpoint (macOS)

Raw evidence from a live troubleshooting session. Partner hostname and IP
redacted because this vault is committed to a public remote; keep the
pattern, not the target.

## Environment

- macOS (Darwin 25.5.0, arch Apple Silicon)
- Zscaler Client Connector installed with admin-locked uninstall passcode
  (employer-deployed on personally-owned hardware) — Z-Tunnel active at
  `utun4 = 100.64.0.1`
- FortiClient VPN present but **disconnected** at time of incident
  (irrelevant to this failure — the second agent doesn't need to be
  running for the routing conflict to manifest)
- Home LAN: `en0 = 192.168.68.55`, default gateway `192.168.68.1`
- Target: partner SFTP endpoint (public hostname → AWS us-west-2 elastic
  IP, both redacted) on custom port `2233`

## Reported symptom

> "can you try to ssh here `<partner-sftp-hostname>` port 2233"

Initial probe from the client:

```
ssh -p 2233 -o BatchMode=yes -o ConnectTimeout=10 <partner-sftp-hostname>
ssh: connect to host <partner-sftp-hostname> port 2233: Connection refused

nc -zv -w 5 <partner-sftp-hostname> 2233
nc: connectx to <partner-sftp-hostname> port 2233 (tcp) failed: Connection refused

nc -zv -w 5 <partner-sftp-hostname> 22
nc: connectx to <partner-sftp-hostname> port 22 (tcp) failed: Connection refused
```

## New observation vs. 2026-09-12 incident

The symptom shape differed from the FortiClient IPsec case documented
in [[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]]:

| 2026-09-12 (FortiClient IPsec) | 2026-09-15 (this SFTP) |
| --- | --- |
| IKE UDP 500/4500 sunk into Z-Tunnel | TCP 2233 sunk into Z-Tunnel |
| Phase-1 **timeout** (`phase1 negotiation failed due to time up.`) | Fast **connection refused** (RST) |
| No response — packets vanished | Immediate RST from ZCC |
| Every port on the destination | Every port on the destination |

**Hypothesis** (not yet independently confirmed against Zscaler docs):
when the hijacked destination is **not** in any ZPA Application Segment
and ZCC's policy has no rule allowing it, ZCC synthesizes a TCP RST
rather than dropping. This is why `route -n get` still shows the tunnel
route, but `nc` returns "refused" instead of hanging until timeout. For
UDP (IKE), there's no RST equivalent, so the traffic just disappears.

If true, RST-vs-timeout is a useful **fingerprint for policy scope**:
- Timeout → destination is in the hijack set, ZCC is trying to tunnel it
- Fast refused → destination is on a hijacked route but ZCC has no rule

## Diagnosis

Followed the exact 30-second check from
[[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] §Fix.1:

```
route -n get <partner-sftp-ip>
```

Output confirmed the hijack:

```
    gateway: 100.64.0.1
  interface: utun4
```

`interface: utun4` (Z-Tunnel) instead of `en0` = client-side tunnel is
intercepting.

## Fix applied (temporary local override)

Same procedure as 2026-09-12:

```
sudo route -n delete <partner-sftp-ip>
sudo route -n add -host <partner-sftp-ip> -gateway 192.168.68.1

route -n get <partner-sftp-ip>
    gateway: 192.168.68.1
  interface: en0

nc -zv -w 5 <partner-sftp-ip> 2233
Connection to <partner-sftp-ip> port 2233 [tcp/infocrypt] succeeded!
```

Port 2233 opened immediately after the route was forced onto `en0`.
Cause confirmed: **Zscaler ZPA was hijacking the route**; the SFTP
service was not down or firewalled.

## Durable fix (not yet executed)

IT ticket to add the partner SFTP hostname / IP to the ZPA
**Application Bypass** list. Same fix path as the 2026-09-12 FortiGate
case — worth referencing that prior ticket when filing this one.

## Open question — which side should the policy live on?

Whether this destination belongs in the **bypass list** (third-party
service reached over the internet) or in the **App Segments** (corp
resource reached via ZPA) depends on the business relationship. Ops
hat: if this is a supplier's endpoint used for legitimate work, IT may
want it *inside* ZPA rather than bypassed — which changes the ticket
ask entirely. Confirm with IT before requesting a bypass.

## Related

- [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] — the
  canonical solution page this evidence strengthens
- [[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]] — the first
  lived incident of this pattern
