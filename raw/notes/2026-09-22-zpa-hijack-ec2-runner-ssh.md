---
type: source
status: stable
created: 2026-09-22
tags: [vpn, zscaler, macos, routing, ssh, aws, ec2, incident, ztna, ci-runner]
---

# Third confirmed Zscaler ZPA route hijack — AWS EC2 CI runner over SSH (macOS)

Raw evidence from a live troubleshooting session. Employer-owned EC2 IP
and instance identifier redacted because this vault is committed to a
public remote; keep the pattern, not the target.

## Environment

- macOS (Darwin 25.5.0, Apple Silicon)
- Zscaler Client Connector active — Z-Tunnel at `utun4 = 100.64.0.1`
  (same as 2026-09-12 and 2026-09-15 incidents)
- FortiClient present but **disconnected** at time of incident
- Public egress via ZCC observed as `167.103.66.184` at test time
- Target: employer CI runner on AWS EC2, `us-west-2`, TCP 22
  (public IP and instance ID redacted)

## Reported symptom

> "ssh on `<runner-ip>` using ubuntu and cybersoft ssh key … check what's
> the issue of the runner"

Initial probes from the client:

```
ssh -i ~/.ssh/certs_cybersoft/cybersoftbpo_deployer \
    -o StrictHostKeyChecking=no -o ConnectTimeout=10 \
    ubuntu@<runner-ip>
ssh: connect to host <runner-ip> port 22: Operation timed out

nc -zv -w 5 <runner-ip> 22
nc: connectx to <runner-ip> port 22 (tcp) failed: Operation timed out

ping -c 2 -W 2 <runner-ip>
Request timeout for icmp_seq 0
100.0% packet loss

traceroute -m 8 -w 2 <runner-ip>
 1  * * *
 2  * * *
 …  (all hops silent)
```

## Diagnosis

30-second check per
[[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] §Fix.1:

```
route get <runner-ip>
route to: ec2-<redacted>.us-west-2.compute.amazonaws.com
destination: 32.0.0.0
       mask: 224.0.0.0
    gateway: 100.64.0.1
  interface: utun4
```

`interface: utun4`, `gateway: 100.64.0.1` — same CGNAT fingerprint as
the prior two incidents. ZCC is hijacking the route to arbitrary
`us-west-2` EC2 addresses (destination pattern `32.0.0.0/3` covers a
huge slice of the internet including most AWS ranges).

Symptom shape this time was **silent timeout** on both ICMP and TCP —
matches the "in ZCC tunnel scope, other side doesn't answer" fingerprint
from the 2026-09-15 note (§Timeout vs. Connection refused). Not the
fast-RST shape.

## Additional variable — EC2 security group

Unlike the partner SFTP case (2026-09-15), the destination here is
employer-owned infrastructure whose security group likely restricts
SSH to specific corp NATs. Even if the ZPA hijack is bypassed, the
ZCC public egress (`167.103.66.184`) may still not be allow-listed.
Two possible failure modes stacked:

1. ZPA hijacks the route → no packets reach EC2 at all (this is what
   the diagnostics show)
2. Even if bypassed, the EC2 security group may reject the personal
   egress IP → would present as timeout post-bypass

So the resolution order matters: fix ZPA scoping first (or use
FortiClient IPsec so egress goes through corp NAT), then confirm SG.

## Fix (not yet executed — session ended before action)

Options presented to user, in speed order:
- Quickest test: toggle Zscaler off, retry SSH.
- If SSH still fails: connect FortiClient IPsec so egress is corp NAT.
- If both fail: verify the EC2 instance is running (stop/start
  reassigns the public IP) and its SG rules.

## Why this matters

This is the **third** time the ZPA-hijacks-arbitrary-destinations
pattern has hit this machine in ~10 days, against three unrelated
targets:

| Date | Destination | Owner | Symptom shape |
| --- | --- | --- | --- |
| 2026-09-12 | Corp FortiGate (IPsec) | Employer | UDP IKE timeout |
| 2026-09-15 | Partner SFTP (TCP 2233) | Third party | Fast TCP RST |
| 2026-09-22 | Employer CI runner (SSH 22) | Employer | Silent TCP + ICMP timeout |

Per the canonical solution page's **third-occurrence rule**, the ask
to IT is no longer another per-host bypass — it's a **ZPA policy
scope review**. ZCC is intercepting far more of the public internet
than legitimate ZPA App Segments require.

## Related

- [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] — canonical solution
- [[raw/notes/2026-09-12-forticlient-ipsec-vs-zscaler]] — 1st incident
- [[raw/notes/2026-09-15-zpa-hijack-partner-sftp]] — 2nd incident
