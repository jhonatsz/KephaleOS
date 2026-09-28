---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-8, proxmox-cluster, identity, active-directory, dc01, dc02]
confidence: high
---

# Phase 8 — Second Proxmox Node + Identity (DC01/DC02)

## Objective

Add a **second Proxmox host** (`pve-01`) to form a **cluster** with `pve-t14-01`. Deploy **two Domain Controllers** (DC01, DC02) on **different physical hosts** — this is when DC redundancy actually means something.

## Why now

Two DCs on one physical host = one power cord away from an outage. Wait until you have two hosts. Also, AD DNS integration is much cleaner when internal DNS (Phase 3) is stable and you know exactly which zones it owns.

## Prerequisites

- Phases 0–7 complete
- Storage (Phase 7) provides shared NFS for cluster-friendly VM storage
- Internal DNS working and stable (Phase 3)
- Comfort with Proxmox already (Phase 0's practice pays off here)

## Hardware

| Item | Required? | Buy timing | Candidates | Rough cost |
|---|---|---|---|---|
| **Second Proxmox host** — 32+ GB RAM, 8+ cores, 1× NVMe (500 GB min) + 1–2× SATA/SAS for local storage | **Required** | BUY NOW | Lenovo ThinkStation P520 (used, ~$400) · Lenovo M720q/M920q + eGPU-less variants · custom AM4/AM5 build · Dell OptiPlex micro-form-factor cluster candidates | US$400–1,200 |
| Additional RAM for T14 | Optional | conditional | conditional on lab RAM headroom | — |
| Second NIC (for cluster corosync separation) | Recommended | BUY NOW | any 1 GbE PCIe or USB — corosync prefers dedicated latency-stable path | US$20 |

## Software

- **Proxmox VE** matching version of `pve-t14-01`
- **Samba AD DC** (recommended, Linux-native, free, RFC-compliant) *or* **Windows Server** (evaluation license for lab; canonical AD experience but licensing pain long-term)

## Network changes

- New host: `pve-01.oikos.home.arpa` on VLAN 10 (mgmt) at `172.27.10.10/24`
- New VMs:
  - `dc01.oikos.home.arpa` on VLAN 20 at `172.27.20.10` (on `pve-t14-01`)
  - `dc02.oikos.home.arpa` on VLAN 20 at `172.27.20.11` (on `pve-01`)
- Cluster corosync: dedicated small subnet if using a second NIC (e.g., `172.27.11.0/24` for cluster traffic)
- Firewall: DC ports allowed from all VLANs (broad — see below)

## Implementation sequence

### Cluster first

1. **Prep pve-01**: install Proxmox with same settings as `pve-t14-01`, register in DNS, add SSH keys.
2. **Storage**: mount the same NFS from `nas01` on both hosts (`tank/vm-storage`).
3. **Create cluster** on `pve-t14-01`: `pvecm create oikos-cluster` (choose a corosync-dedicated interface if you have one).
4. **Join** `pve-01` to the cluster: `pvecm add <pve-t14-01-ip>`.
5. **Verify** — `pvecm status` shows two nodes, quorum OK.
6. **Test live migration** of a small VM between the two hosts.

### Then identity

7. **DNS delegation planning** — decide whether AD DNS will *replace* AdGuard for `oikos.home.arpa` (recommended: keep AdGuard for filtering + client queries; forward `oikos.home.arpa` to DCs). Record decision in an ADR.
8. **Deploy DC01**:
   - Debian 12 or Ubuntu 22.04 base (or Windows Server if you chose it)
   - `samba-tool domain provision --realm=OIKOS.HOME.ARPA --domain=OIKOS --dns-backend=SAMBA_INTERNAL`
   - Assign static IP `172.27.20.10`, hostname `dc01.oikos.home.arpa`
9. **Configure DNS**:
   - AdGuard: conditional forwarder — `oikos.home.arpa` → `172.27.20.10, 172.27.20.11`
   - DCs answer for `_msdcs.oikos.home.arpa` etc.
10. **Deploy DC02** on `pve-01`:
    - Provision as additional DC: `samba-tool domain join`
    - Verify replication: `samba-tool drs showrepl`
11. **First user + group**:
    - Create `Domain Admins` service accounts (not root-equivalent daily-driver accounts)
    - Create `oikos.admins` for infra admins
    - Create yourself as a non-admin daily-driver user
12. **Join a member server** (e.g., `obs01`) to the domain to prove Kerberos + LDAP work.
13. **Group Policy** (Windows Server route only) or **Samba DFS + login scripts** (Samba route).
14. **Backups** — AD-aware backup: `samba-tool domain backup online --targetdir=...` written to NAS.
15. **Monitoring** — Prometheus scrapes DCs (samba_exporter or windows_exporter); alerts on replication failure.

## CCNA topics reinforced

- DNS delegation and stub zones
- Kerberos protocol basics (not strictly CCNA but adjacent)
- LDAP over TLS
- SRV records (AD depends heavily on them: `_ldap._tcp`, `_kerberos._udp`)
- Time synchronization criticality (Kerberos fails if clock skew > 5 min)

## Study track lab

Not directly CCNA — but worth building an OSPF + DNS delegation lab in EVE-NG to prove the concept end-to-end.

## Physical Oikos lab exercise

- Live-migrate a running VM between `pve-t14-01` and `pve-01`; observe zero-second downtime (with shared NFS storage)
- Shut down DC01 — join a new machine to the domain — should still work via DC02
- On any Linux VM: `kinit user@OIKOS.HOME.ARPA` — get a ticket — `klist`
- `dig SRV _ldap._tcp.dc._msdcs.oikos.home.arpa` — should return DC01 + DC02

## Break/fix drill

1. Power-off `pve-t14-01`. Cluster loses quorum (2-node with no witness). Understand the "expected votes = 2" problem; discuss with future-you whether to add a **QDevice** on nas01 for 3-way quorum.
2. Simulate DC01 disk failure: shut down `dc01`, force replication conflicts on DC02, then restore from backup. Verify domain still functional throughout.
3. Break DNS delegation — remove AdGuard's forwarder for `oikos.home.arpa`. Observe: domain-join fails. Restore.

## Verification commands

Proxmox cluster:

```bash
pvecm status
pvecm nodes
ha-manager status
```

DC (Samba):

```bash
samba-tool drs showrepl
samba-tool domain info 127.0.0.1
samba-tool user list
samba-tool group list
kinit administrator@OIKOS.HOME.ARPA
klist
```

Client:

```bash
realm discover oikos.home.arpa
realm join oikos.home.arpa -U administrator
id user@oikos.home.arpa
```

## Security posture at this phase

- DC service accounts: non-daily-driver, unique per service, in password manager
- Kerberos ticket lifetime: 10 h default; renewable 7 days
- LDAP over TLS mandatory (LDAPS on 636)
- SMB signing required
- **Time**: DCs must reference the same NTP source (chrony on OPNsense or a dedicated time server); alert on skew > 30 s
- Audit logging on both DCs → Loki
- Restrict `Domain Admins` membership to break-glass accounts only

## Monitoring coverage

- Cluster corosync ring health
- Node uptime + resource pressure
- AD replication status (must succeed every 15 min)
- Kerberos ticket-request success rate
- LDAP query latency
- Backup age of `samba-tool domain backup`

## Kephaleos docs to create / update

- **Create**: `runbooks/proxmox-cluster.md`
- **Create**: `runbooks/proxmox-add-node.md`
- **Create**: `runbooks/samba-ad-provision.md`
- **Create**: `runbooks/samba-ad-add-dc.md`
- **Create**: `runbooks/domain-join-linux.md`
- **Create**: `runbooks/ad-backup-restore.md`
- **Update**: `current-state.md`, `inventory/nodes.md` (`pve-01`, `dc01`, `dc02`), `architecture/address-plan.md`
- **New ADRs**: `adr/000N-cluster-quorum-strategy.md`, `adr/000N-identity-platform.md`, `adr/000N-dns-delegation-adguard-vs-dc.md`
- **Append**: `CHANGELOG.md`

## Diagram updates

- Master diagram: two hosts, cluster line, DCs
- Identity/DNS diagram (new): clients → AdGuard → DC forwarders
- Service dependency diagram: update

## Completion checklist

- [ ] `pve-01` installed and cluster-joined
- [ ] Shared NFS storage mounted on both nodes
- [ ] Live migration tested
- [ ] Cluster quorum strategy decided + documented (QDevice or manual)
- [ ] DC01 provisioned, first user created
- [ ] DC02 added as additional DC, replication verified
- [ ] AD DNS delegation from AdGuard configured
- [ ] At least one Linux member server successfully joined
- [ ] Kerberos ticket acquisition tested
- [ ] AD backup + restore tested
- [ ] Monitoring covers cluster + DCs
- [ ] Docs, ADRs, diagrams updated
