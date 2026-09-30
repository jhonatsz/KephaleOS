---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-7, storage, nas, zfs, backup]
confidence: high
---

# Phase 7 — Storage (NAS + Backups)

## Objective

Deploy a dedicated NAS with **ZFS** (or Btrfs), providing SMB (family) + NFS (Proxmox / VMs) storage. Establish a **3-2-1 backup regime** (3 copies, 2 media, 1 off-site).

## Why now

Everything from here up (identity, k8s, AI) needs shared persistent storage. And the household deserves a resilient place for photos/documents. Doing it before k8s (Phase 9) means k8s can consume it via NFS/iSCSI from day 1.

## Prerequisites

- Phases 0–5 complete (VLAN 40 will host storage; monitoring already covers it)
- Off-site backup target chosen (Backblaze B2 / AWS S3 Glacier Deep Archive / rotating USB drive at a relative's / friend's house)
- Bandwidth budget for backups if using cloud

## Hardware

| Item | Required? | Buy timing | Candidates | Rough cost |
|---|---|---|---|---|
| **NAS chassis** (4-bay minimum, 6-bay preferred) | **Required** | BUY NOW | Synology DS923+ / DS1522+ · TrueNAS Mini X · custom build (Fractal Node 304 + ASRock J5040 or N100 board) | US$400–1,200 |
| **NAS drives** — 4× 4–8 TB CMR (not SMR), NAS-grade | **Required** | BUY NOW | WD Red Plus · Seagate IronWolf · Toshiba N300 | US$100–200 each |
| **Cold-spare drive** | Recommended | BUY NOW | one extra of same model | 1× drive cost |
| **NVMe cache** (optional for Synology / TrueNAS) | Optional | BUY WAIT | any NAS-rated NVMe | US$60–120 |
| **Off-site backup drive** or cloud plan | **Required** | BUY NOW | USB 3 external 8+ TB · or Backblaze B2 | US$120 + power |
| UPS (from Phase 5) | Required | — | Reuse or expand VA | — |
| 2.5 GbE or 10 GbE NIC upgrade | Optional | conditional | conditional on switch support | — |

## Software

- **TrueNAS SCALE** (Debian-based, k8s-native, includes docker apps) *or* **Synology DSM** (locked to Synology hardware but very polished) *or* **OpenMediaVault** (Debian + Btrfs, DIY)
- **restic** or **rustic** for backup engine — encrypted, deduplicated, incremental
- **rclone** for cloud sync
- **Sanoid + Syncoid** for ZFS snapshot management + replication (if TrueNAS)

## Network changes

- New host: `nas01.oikos.home.arpa` on VLAN 40 (storage), IP e.g. `172.27.40.10/24`
- Firewall rules — storage VLAN policy:
  - `servers` → `nas01`: allow NFS (2049) + SMB (445)
  - `k8s` (future) → `nas01`: allow NFS + iSCSI (3260)
  - `ai` (future) → `nas01`: allow NFS
  - `home` → `nas01`: allow SMB
  - `nas01` → Internet: allow HTTPS (rclone to cloud) + NTP; deny everything else
  - Everything else → `nas01`: deny
- Optional: jumbo frames (MTU 9000) on VLAN 40 — requires switch + all clients supporting it (revisit at deployment)

## Implementation sequence

1. **Pool design**:
   - 4× 6 TB in RAIDZ2 (2 parity, ~12 TB usable) — recommended balance
   - Or 6× 6 TB in RAIDZ2 (~24 TB usable)
   - Or mirror pairs (better performance, lower usable capacity)
2. **Rack + power-up** — connect to VLAN 40 access port
3. **Install NAS OS** — TrueNAS or DSM
4. **Create pool + datasets**:
   - `tank/vm-storage` — NFS export for Proxmox
   - `tank/family` — SMB share, photo/doc storage
   - `tank/backup-target` — dataset for restic/rustic repos from other hosts
   - `tank/media` — SMB share for media
5. **NFS exports** — Proxmox as authorized client; Kerberos deferred
6. **SMB shares** — family users, ACLs, macOS Time Machine dataset if applicable
7. **Snapshots**:
   - Hourly (24 kept)
   - Daily (30 kept)
   - Weekly (8 kept)
   - Monthly (12 kept)
8. **Off-site backup**:
   - restic repo on off-site USB (rotated weekly) *or*
   - restic → Backblaze B2 (nightly, encrypted)
   - Backup all datasets + Proxmox config + OPNsense config + Kephaleos + homelab repo
   - **Test a restore** monthly — untested backup = no backup
9. **Proxmox integration**:
   - Add NFS storage in Proxmox (Datacenter → Storage → NFS)
   - Migrate existing VM disks from local LVM-thin to NFS if desired (or keep some local, some remote)
10. **Monitoring** — add nas01 to Prometheus scrape targets:
    - node_exporter (system)
    - zfs_exporter (pool health, ARC hit ratio, scrub status)
    - SMART attributes via smart_exporter
    - Alerts: any drive `FAULTED`, pool `DEGRADED`, capacity > 80%, scrub failed
11. **Scrub schedule** — monthly minimum

## CCNA topics reinforced

- NFS / SMB port + protocol behavior
- MTU / jumbo frames tradeoffs
- Storage traffic isolation (why VLAN 40 exists)
- QoS considerations (defer to Phase 11 unless painful)

## Study track lab

Nothing new specifically — but if using iSCSI, worth a Packet Tracer / EVE-NG detour into subnetting per-storage-VLAN.

## Physical Oikos lab exercise

- Simulate a drive failure: pull one drive from the array (with the pool safe); observe `zpool status` → DEGRADED; reinsert; observe resilver
- From a Proxmox host: `mount -t nfs nas01:/tank/vm-storage /mnt/nas-test`; write 10 GB; measure throughput; unmount
- From your workstation: mount SMB share; write; read; delete; verify snapshot brings it back

## Break/fix drill

1. `zpool export tank` on the NAS. Observe: NFS mounts on Proxmox hang. Recover with `zpool import tank`.
2. Fill `tank/family` to > 95%. Observe: SMB write failures. Understand ZFS's 80% rule. Free space.
3. Rotate the off-site USB drive on schedule; **verify restic can read the previous one** before writing new snapshots.

## Verification commands

NAS:

```bash
zpool status
zpool list
zfs list -o name,used,avail,mountpoint,compression,compressratio
zfs get all tank/family
smartctl -a /dev/sda
restic snapshots --repo /mnt/backup-usb
```

Client:

```bash
showmount -e nas01                # NFS export list
mount -t nfs nas01:/tank/vm-storage /mnt/x
dd if=/dev/zero of=/mnt/x/test bs=1M count=1000    # write throughput
smbclient -L //nas01 -U <user>    # SMB share list
```

## Security posture at this phase

- NAS admin UI: HTTPS + strong password + LAN-side only + 2FA if supported
- SMB: SMB3 minimum; SMB1 disabled
- NFS: v4 with Kerberos preferred (deferred to Phase 8 when AD/Kerberos exists), v3 with IP-based export ACLs otherwise
- ZFS **encryption at rest** if drives leave the house (Backblaze B2 always encrypted client-side via restic)
- Backup encryption key: escrowed in password manager + one printed physical copy in a safe
- Camera VLAN NEVER writes to family datasets

## Monitoring coverage

- ZFS pool health, capacity, scrub, ARC hit ratio
- SMART per drive
- NFS/SMB throughput
- Backup: last successful snapshot age (alert if > 26 h)
- Snapshot count (alert if outside expected range)

## Kephaleos docs to create / update

- **Create**: `runbooks/nas-baseline.md`
- **Create**: `runbooks/zfs-pool-design.md`
- **Create**: `runbooks/backup-and-restore.md`
- **Create**: `backup/backup-policy.md` — 3-2-1 policy, retention, restore SLAs
- **Create**: `backup/restore-drills-log.md` — dated log of restore drills (do one monthly)
- **Update**: `current-state.md`, `inventory/nodes.md` (`nas01`), `architecture/address-plan.md`
- **New ADR**: `adr/000N-nas-platform.md`, `adr/000N-backup-strategy.md`
- **Append**: `CHANGELOG.md`

## Diagram updates

- Storage diagram (new): NAS → VLANs → consumers
- Backup dependency diagram: sources → NAS → off-site

## Completion checklist

- [ ] NAS hardware installed on VLAN 40
- [ ] ZFS pool created (RAIDZ2 or chosen topology)
- [ ] All datasets created with compression + snapshots
- [ ] NFS export to Proxmox working
- [ ] SMB shares working from family devices
- [ ] Snapshot schedule active
- [ ] Off-site backup working (restic verified)
- [ ] **Restore drill completed** — recover a file, then a full dataset
- [ ] Monitoring + alerts wired in
- [ ] Docs + ADRs + diagrams updated
