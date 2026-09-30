---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-5, observability, prometheus, grafana, monitoring]
confidence: high
---

# Phase 5 — Observability Foundation

## Objective

Metrics, logs, and alerts covering everything that exists so far (OPNsense, switch, APs, DNS, Proxmox host, VMs) — plus every service added from here on. Grafana dashboards, Prometheus alerting, syslog aggregation.

## Why now

Half your infrastructure is currently a black box. Adding observability *now* — before storage, identity, k8s, and AI — means every subsequent phase lands with monitoring built-in, not bolted on. Adding it later means every phase has a "we should really monitor this" tech-debt line.

## Prerequisites

- Phases 0–3 complete (something to monitor, working DNS, VLANs isolating mgmt)
- 8 GB free RAM on Proxmox for the monitoring VM
- 100 GB free disk space (Prometheus TSDB grows; retention policy set below)

## Hardware

| Item | Required? | Buy timing | Notes |
|---|---|---|---|
| **UPS** | Recommended (if not already) | BUY NOW | Rack or line-interactive 1000–1500 VA · APC Back-UPS Pro 1500 / CyberPower CP1500PFCLCD. Sized for firewall + switch + Proxmox + NAS-headroom. Monitored via NUT (Phase 5 wires it up) |
| — | — | — | Everything else runs as VMs |

## Software

- **Prometheus** — metrics, 30-day retention initially
- **Grafana** — dashboards
- **Loki** + **Promtail** — logs (light on RAM vs. ELK)
- **Alertmanager** — routes alerts to email + optional Slack/Telegram
- **Exporters**:
  - `node_exporter` on every Linux host
  - `snmp_exporter` for switch + APs (SNMPv3)
  - `blackbox_exporter` for HTTPS probes
  - `proxmox_ve_exporter` for Proxmox cluster metrics
  - `opnsense_exporter` (or telegraf + inputs.opnsense)
  - `adguard_exporter`
- **NUT** — UPS monitoring on Proxmox host

## Network changes

- New VM: `obs01.oikos.home.arpa` on VLAN 20 (servers), `172.27.20.30`
- Firewall: allow VLAN 20 → all other VLANs on scrape ports (9100, 9116, etc.)
- Allow every host → `obs01` on 3100 (Loki push)

## Implementation sequence

1. **Provision VM** — Debian 12, 4 vCPU, 8 GB RAM, 100 GB disk on VLAN 20.
2. **Install Prometheus + Grafana + Alertmanager + Loki** — via docker-compose or Ansible role (whichever you've practiced first, per invariant 4). Store compose in the homelab repo.
3. **Deploy node_exporter everywhere** — Proxmox host, every VM. Ansible playbook.
4. **snmp_exporter** — set up SNMPv3 users on the switch and APs; add scrape targets in Prometheus.
5. **opnsense_exporter or telegraf** — install on OPNsense (via plugin) or scrape via SNMP.
6. **proxmox_ve_exporter** — one instance, points at Proxmox API.
7. **adguard_exporter** — scrape AdGuard's stats endpoint.
8. **blackbox_exporter** — HTTPS/HTTP probes for OPNsense UI, Proxmox UI, AdGuard UI, external `1.1.1.1`.
9. **Dashboards** — start with Grafana community dashboards (Node Exporter Full, Proxmox, OPNsense, AdGuard) then customize.
10. **Loki + Promtail** — Promtail on every Linux host; syslog forwarding from OPNsense, switch, APs to Loki (rsyslog or vector).
11. **Alerts** — bootstrap set:
    - Host down > 2 min
    - Disk > 85% full
    - Memory > 90% for 10 min
    - CPU > 85% for 10 min
    - Certificate expiry < 14 days
    - OPNsense WAN down
    - AdGuard down
    - UPS on battery > 30 s
12. **NUT for UPS** — Proxmox host as NUT server, VMs as clients; alert on battery-below-30%.
13. **Backup Grafana + Alertmanager configs** to git.

## CCNA topics reinforced

- SNMP (v3 auth+priv, community strings vs. users)
- Syslog levels and facilities
- Network monitoring paradigms (pull vs. push)
- NetFlow/sFlow concepts (deferred — Phase 6+ maybe)

## Study track lab

Not directly reinforced by Packet Tracer, but worth practicing SNMPv3 config in EVE-NG once Phase 6 exists. For now: read Cisco SNMPv3 config chapter of the Odom guide.

## Physical Oikos lab exercise

- Open Grafana, click through every dashboard, understand every panel
- Trigger a test alert (stop `node_exporter` on one host); confirm delivery
- Query Loki for OPNsense firewall drops in the last 5 minutes: `{job="opnsense"} |= "block in"`
- Simulate UPS on-battery (press UPS test button); watch NUT alert fire

## Break/fix drill

1. Fill `/var/lib/prometheus` with junk to 90%. Observe: disk-full alert, then Prometheus stops writing. Recover.
2. Kill Grafana. Confirm alerts still fire (Alertmanager independent).
3. Rotate SNMP credentials without updating Prometheus — watch scrapes fail — restore.

## Verification commands

```bash
# on obs01
docker ps                        # or systemctl status prometheus grafana alertmanager loki
curl -s localhost:9090/-/healthy
curl -s localhost:3000/api/health
promtool check config /etc/prometheus/prometheus.yml
amtool check-config /etc/alertmanager/alertmanager.yml
logcli query '{job="opnsense"}' --limit 20    # Loki CLI

# from a scrape target
curl -s localhost:9100/metrics | head
```

## Security posture at this phase

- Grafana behind TLS + strong admin password; SSO deferred
- Prometheus + Alertmanager: no auth (network-isolated), reachable only from mgmt VLAN
- SNMPv3 auth+priv (no v2c anywhere in production)
- Loki: same isolation as Prometheus

## Monitoring coverage

Meta: this phase *is* the monitoring coverage. Verify every existing host/service has:

- [ ] node_exporter or equivalent metrics
- [ ] Log shipping to Loki
- [ ] At least one dashboard
- [ ] At least one alert

## Kephaleos docs to create / update

- **Create**: `runbooks/prometheus-baseline.md`, `runbooks/grafana-baseline.md`, `runbooks/loki-baseline.md`
- **Create**: `monitoring/alert-catalog.md` — every alert, what it means, what to do
- **Create**: `monitoring/dashboards.md` — index of dashboards with links
- **Create**: `monitoring/slis-and-slos.md` (optional but recommended)
- **Update**: `current-state.md`, `inventory/nodes.md` (add `obs01`), `architecture/address-plan.md`
- **Append**: `CHANGELOG.md`

## Diagram updates

- Add observability layer to master diagram
- Service dependency diagram: what monitors what

## Completion checklist

- [ ] `obs01` running Prometheus, Grafana, Alertmanager, Loki
- [ ] node_exporter on every Linux host
- [ ] SNMP scrapes working for switch + APs
- [ ] OPNsense metrics + logs flowing
- [ ] AdGuard metrics + logs flowing
- [ ] UPS monitored via NUT; battery/on-line status in Grafana
- [ ] Alert catalog: at least the 8 bootstrap alerts, tested end-to-end
- [ ] All configs in git
- [ ] Docs + diagrams updated
