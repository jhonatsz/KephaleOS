---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-3, dns, dhcp, ntp, adguard, network-services]
confidence: high
---

# Phase 3 — Network Services (DNS, DHCP, NTP, Filtering)

## Objective

Stand up **internal DNS** (authoritative for `oikos.home.arpa`), **DHCP scopes with option 15** (domain-name) pushing every client onto the internal zone, **NTP** for time sync, and **DNS filtering** (ad/malware blocking) household-wide.

## Why now

You have VLANs (Phase 2), and every service you'll add from now on wants: (a) a resolvable name, (b) a stable address, (c) accurate time. Doing this once now saves rework across every future phase.

## Prerequisites

- Phase 2 complete (VLANs live, mgmt reachable, DHCP working per-VLAN on OPNsense)
- ADR oikos-0001 (namespace `oikos.home.arpa`)
- One VM slot free on the T14 Proxmox (or two, if separating DNS and filter)

## Hardware

| Item | Required? | Buy timing | Notes |
|---|---|---|---|
| — | — | none | Runs as VMs on existing Proxmox |
| UPS | Recommended | BUY SOON | DNS outage = household internet perceived-outage; power protection matters here |

## Software

- **AdGuard Home** — DNS filtering + local records + upstream (recommended over Pi-hole; better UI, DoH/DoT support, easier local records). Or Pi-hole if you prefer.
- **Unbound** — recursive resolver upstream (already on OPNsense, optionally standalone if you want AdGuard to forward to it)
- **chrony** — NTP server on a small VM (or use OPNsense's built-in NTP)
- Debian 12 template (from Phase 0)

## Network changes

- New VMs on VLAN 20 (servers):
  - `dns01.oikos.home.arpa` — `172.27.20.20` — AdGuard Home
  - Optional `dns02` for redundancy at `172.27.20.21`
- OPNsense DHCP scopes updated: **DNS servers** point at `172.27.20.20` (and `.21` if present)
- OPNsense DHCP option 15 (domain-name): `oikos.home.arpa`
- OPNsense DHCP option 42 (NTP): `172.27.20.20` (or OPNsense LAN IP)
- Firewall rule: all VLANs allowed to reach VLAN 20 on UDP/TCP 53 + UDP 123 only

## Implementation sequence

1. **Build the DNS VM** — clone the Debian template, hostname `dns01`, static IP `172.27.20.20/24`, gateway `172.27.20.1`.
2. **Install AdGuard Home** — official install script, admin UI on `:3000` initially, move to `:443` after cert.
3. **Configure AdGuard**:
   - Listen on port 53 (all interfaces)
   - Upstream: `https://1.1.1.1/dns-query` (DoH) + `https://9.9.9.9/dns-query`, parallel resolution
   - Bootstrap DNS: `1.1.1.1`, `1.0.0.1`
   - Cache: 4 GB max, 3600 s min TTL
   - Filters: OISD Big, HaGeZi Multi Pro, StevenBlack, plus AdGuard defaults
   - Local records for `oikos.home.arpa` zone (add every host manually until Phase 8 introduces AD DNS)
4. **Configure OPNsense DHCP**:
   - Services → DHCPv4 → LAN (VLAN 100): DNS servers = `172.27.20.20`; Domain = `oikos.home.arpa`
   - Repeat for every VLAN
   - NTP option: `172.27.20.20` (or OPNsense IP)
5. **Verify** — release/renew DHCP on a family device; `nslookup pve-t14-01.oikos.home.arpa` should resolve.
6. **NTP** — install chrony on `dns01` (or reuse OPNsense NTP); confirm `chronyc sources`.
7. **Split-horizon check** — `nslookup google.com` still works (upstream); `nslookup dns01.oikos.home.arpa` resolves locally.
8. **Add local records** for every device currently known (T14, OPNsense, switch, APs when they exist).
9. **Optional dns02** — second VM, same config (import AdGuard config); OPNsense DHCP pushes both.
10. **Document** — runbook, current-state, changelog, and the internal-DNS diagram.

## CCNA topics reinforced

- DNS resolution flow: stub → recursive → authoritative
- DHCP options (15, 42, 6)
- DHCP relay (`ip helper-address`) — study track Lab 06 preview
- NTP stratum concept
- Split-horizon DNS
- Firewall rule scoping (allow narrow ports only)

## Study track lab

- **Lab 05**: static + default routes across two routers (foundational for understanding why DNS servers need to be reachable across VLANs)
- **Lab 06**: DHCP relay across a router — set up one central DHCP scope, `ip helper-address` on remote VLANs. Mirrors OPNsense's centralized DHCP model.

## Physical Oikos lab exercise

- On your workstation: `dig @172.27.20.20 pve-t14-01.oikos.home.arpa` — should return the mgmt IP
- On the same client: `dig google.com` — should hit AdGuard, get filtered, then upstream
- On a family device: try to load a known-blocked ad domain (e.g. `doubleclick.net`) — should be NXDOMAIN or 0.0.0.0
- On any VLAN: `chronyc sources` — one accurate stratum

## Break/fix drill

1. Shut down `dns01`. Observe: **what still works?** (IP-only traffic works; anything with hostnames breaks after cache expiry). Notes: this is why dns02 exists.
2. Restart `dns01`; verify recovery.
3. Misconfigure OPNsense DHCP option 15 (typo the domain). Renew a client — observe FQDN resolution behavior. Fix.
4. Point a client at `8.8.8.8` manually (bypassing AdGuard). Explain how you'd detect and block this at the firewall.

## Verification commands

DNS VM:

```bash
systemctl status AdGuardHome
ss -tulnp | grep :53
tail -f /opt/AdGuardHome/data/querylog.json | jq
chronyc sources
chronyc tracking
```

From a client:

```bash
dig @172.27.20.20 pve-t14-01.oikos.home.arpa +short
dig @172.27.20.20 google.com
nslookup shouldbeblocked.example
resolvectl status | grep -A2 "Current DNS"
```

OPNsense:

```bash
configctl dhcpd list
tail -f /var/log/dhcpd.log
```

## Security posture at this phase

- AdGuard admin UI: HTTPS only, strong password, reachable only from mgmt VLAN + workstation
- Firewall: **block direct DNS to Internet** from all VLANs *except* AdGuard (VLAN 20). Every client must resolve through internal DNS.
  - Rule: `!src=172.27.20.0/24` `dst=any` `dport=53` → **deny**
- Prevents DoH bypass? No — DoH goes over 443. Accept this limitation; block known DoH hostnames if you want to be strict (Phase 5's ACL work).
- Camera VLAN: deny Internet — cameras must resolve locally only, break their cloud service on purpose

## Monitoring coverage

- AdGuard's built-in query log + stats (7-day retention)
- Still no Prometheus (Phase 5 wires it up); when Phase 5 lands, add `adguard_exporter` and dashboards

## Kephaleos docs to create / update

- **Create**: `runbooks/adguard-baseline.md`
- **Create**: `runbooks/opnsense-dhcp-per-vlan.md`
- **Create**: `runbooks/dns-add-local-record.md` (quick-reference for adding a host)
- **Update**: `current-state.md`, `inventory/nodes.md` (`dns01` added), `architecture/address-plan.md` (VLAN 20 populated)
- **Append**: `CHANGELOG.md`

## Diagram updates

- Add DNS/DHCP dependency diagram: client → OPNsense DHCP → `dns01` → upstream
- Update master diagram to show `dns01` on VLAN 20

## Completion checklist

- [ ] `dns01` VM running AdGuard Home
- [ ] All VLAN DHCP scopes push `dns01` as DNS server + `oikos.home.arpa` as domain
- [ ] Local zone records cover every existing host
- [ ] Internal FQDN resolution works from every VLAN (mgmt, home, iot, camera, guest, servers)
- [ ] External DNS still resolves (filtered)
- [ ] NTP working — `chronyc sources` shows healthy stratum
- [ ] Firewall blocks direct-to-Internet DNS except from `dns01`
- [ ] Break/fix drill completed
- [ ] Runbooks, diagrams, docs updated
