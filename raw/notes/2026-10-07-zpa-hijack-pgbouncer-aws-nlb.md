---
type: source
status: stable
created: 2026-10-07
tags: [vpn, zscaler, macos, routing, aws, nlb, pgbouncer, incident, cross-account, rds]
---

# ZPA hijack #5 — AWS NLB (eproxy-nprd/prd pgbouncer, TCP 5432)

Fifth Zscaler ZPA FIB-hijack incident on the operator's personal Mac in ~26 days. First time the hijacked destination is **shared AWS infrastructure** (public NLBs fronting cybersoft pgbouncer) rather than a single corp endpoint. Also produced two new side-learnings: a two-egress-IP distinction (direct vs Zscaler-tunneled) that breaks naive AWS SG whitelisting, and a persistent-workaround LaunchDaemon design.

Session: 2026-10-07 → 2026-10-08. Primary task was an `omnibench` DB schema propagation (nprd → prd), but 60%+ of the elapsed time was network-path setup, dominated by this hijack.

## Environment

- Same Mac as incidents #1–#4
- Zscaler Client Connector installed (admin-locked passcode)
- Home LAN: `en0 = 192.168.68.x`
- Destinations:
  - `eproxy-nprd.mis.cybersoftbpo.com` → NLB zonal IPs `32.187.21.89`, `54.148.151.203`
  - `eproxy-prd.mis.cybersoftbpo.com` → NLB zonal IP `32.186.82.239`
  - All in Zscaler's known-hijacked scope `32.0.0.0/3`

## Symptom + diagnosis

Classic Mode A fingerprint:

```
$ nc -zv eproxy-nprd.mis.cybersoftbpo.com 5432
nc: Connection refused   # Mode A1 shape

$ route -n get 32.187.21.89
   interface: utun*
   gateway:   100.64.0.*  ← CGNAT, FIB hijack confirmed
```

Shape **shifted mid-session** from fast RST (A1) to silent timeout (A2) on the prod NLB. Didn't correspond to a diagnosis change — the pgbouncer pods were briefly all unhealthy in the only populated AZ, flipping the NLB from "RST because no policy" to "silent drop because dead tunnel". When the pods came back, so did the RST. First-time observation that A1 ↔ A2 can toggle on the same destination within one session.

## Fix applied (standard §Mode A)

```bash
LAN_GW=$(netstat -nr -f inet | awk '/^default/ {print $2; exit}')
for ip in 32.187.21.89 54.148.151.203 32.186.82.239; do
  sudo route -n delete "$ip"
  sudo route -n add -host "$ip" -gateway "$LAN_GW"
done
```

Worked. Reverted on later Zscaler policy sync as expected.

## New side-learning #1 — Two-egress-IP distinction

Debugging SG whitelist failures revealed the operator has **two distinct public IPs** depending on how traffic exits:

| Path | IP seen |
| --- | --- |
| `curl https://checkip.amazonaws.com` | `49.149.219.16` (direct ISP; AWS IPs are route-overridden) |
| `curl https://ifconfig.me`            | `167.103.64.207` (Zscaler-tunneled path) |

Both IPs are real; both belong to the operator's ISP (PLDT) but on different /24s because the Zscaler path exits through a different ISP endpoint. Which one a destination sees depends on whether the operator route-overrode that destination's IP:

- Route-overridden → direct → AWS sees `49.149.219.16`
- Not overridden → Zscaler tunnel → service sees `167.103.64.207`

**Operational consequence**: for AWS SG whitelisting, the only IP that matters is the direct one. Whitelisting the Zscaler-tunneled IP does nothing — packets to AWS never egress from it because the route-override short-circuits Zscaler. Burned ~30 min adding the wrong IP to nprd + prd SGs before realizing. Both stale `167.103.64.207` /32 rules still sit in those SGs as of session end (cleanup pending).

## New side-learning #2 — LaunchDaemon for persistent route overrides

Current practice (per the deferral decision): run `sudo route delete/add` each time the hijack fires. Dies on reboot, Wi-Fi switch, Zscaler policy refresh — each requires re-running.

This session designed a persistent-workaround macOS LaunchDaemon that:

- Reads destinations from `/etc/zscaler-whitelist.conf` (hostnames or IPs, `#` comments)
- Resolves each hostname via `dig` on every run (handles AWS NLB IP rotation)
- Re-applies `route delete / add -host <ip> -gateway <lan-gw>` on:
  - Boot (`RunAtLoad=true`)
  - Every 15 min (`StartInterval=900`)
  - Every network state change (`KeepAlive.NetworkState=true`)
- Logs to `/var/log/zscaler-whitelist.log`

Staged at `~/.local-staging/zscaler-whitelist/` with `install.sh` / `uninstall.sh`. Not yet installed — operator left it as a durable workaround choice for next time.

This is a **middle path** between manual command (current) and IT escalation (deferred). Lower friction per recurrence than manual; still a workaround, not a structural fix. Fits the deferral decision's spirit: reduce per-incident cost without taking on the IT-coordination cost.

## New side-learning #3 — Cross-account bastion → prod RDS

Separate from Zscaler. The cybersoft msd-admin bastion (`i-0940202357e886b53`, private `10.51.100.54`, VPC `vpc-03f1c849aa87c7a07` in CIDR `10.51.0.0/16`, account `872194582181`) couldn't reach mis-admin prod RDS (`omnibench-prd` at `10.79.238.135`, VPC `vpc-00120a39c43841655` in `10.79.0.0/16`, account `654654370132`), despite:

- VPC peering `pcx-039dfe2c15b4cc7b8` being ACTIVE between the two VPCs
- The bastion's **public** IP `34.222.146.216/32` already being whitelisted on prod RDS SG `sg-01005296daf52db66` — **cosmetic/stale**, because cross-account traffic over peering uses **private** IPs

Required three coordinated changes on prod RDS subnets (dedicated route table + NACL, so blast radius contained to just those 2 subnets):

```bash
# 1. Route — prod RDS subnet route table rtb-01402748eb9e0fb20
aws ec2 create-route --route-table-id rtb-01402748eb9e0fb20 \
  --destination-cidr-block 10.51.0.0/16 \
  --vpc-peering-connection-id pcx-039dfe2c15b4cc7b8

# 2. NACL ingress — acl-0c98c2ff00f0eab61 (rule 140)
aws ec2 create-network-acl-entry --network-acl-id acl-0c98c2ff00f0eab61 \
  --rule-number 140 --protocol tcp --rule-action allow \
  --ingress --cidr-block 10.51.0.0/16 --port-range From=5432,To=5432

# 3. NACL egress — same NACL, ephemeral range
aws ec2 create-network-acl-entry --network-acl-id acl-0c98c2ff00f0eab61 \
  --rule-number 140 --protocol tcp --rule-action allow \
  --egress --cidr-block 10.51.0.0/16 --port-range From=1024,To=65535

# 4. SG — prod RDS sg-01005296daf52db66
aws ec2 authorize-security-group-ingress --group-id sg-01005296daf52db66 \
  --ip-permissions 'IpProtocol=tcp,FromPort=5432,ToPort=5432,IpRanges=[{CidrIp=10.51.100.54/32,...}]'
```

Nprd equivalent "just worked" because nprd RDS SG has `0.0.0.0/0` on 5432 — flagged separately as a security finding (sg-04d5e2eb4e1948f88).

## Lessons

- Zscaler's `32.0.0.0/3` hijack scope is wide enough that most AWS public NLBs resolve into it. Any AWS-fronted shared service accessed from this Mac is a candidate for a future recurrence.
- The two-egress-IP phenomenon makes SG whitelist debugging treacherous — always correlate the IP being whitelisted with the exact path the traffic takes. `curl` from different services can give different answers, and both are "correct."
- A1 ↔ A2 shape can toggle within one destination within one session as backend health changes. Don't re-diagnose from scratch when the shape changes mid-session; it's still Mode A.
- Prod RDS SG whitelist entries with **public** IPs for cross-account bastions are cosmetic — someone added `34.222.146.216/32` thinking it would work; it never has. Easy mistake worth documenting.
- Three-coordinated-change pattern (route + NACL in + NACL out + SG) is the full cross-account RDS access checklist when peering already exists.

## Related

- [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] — canonical; recurrence table updated with this incident
- [[wiki/decisions/2026-10-01-zpa-escalation-deferral]] — deferral still stands; this is #5 but doesn't fire any explicit revisit trigger yet
- Session also added one entry to the operator's claude-code memory for the `cybersoft/pgbouncer` project (feedback_user_ip.md) about the two-IP distinction
