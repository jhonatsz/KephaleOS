---
type: project
status: active
created: 2026-09-15
updated: 2026-09-15
aliases:
  - "CyberSoft SFTP"
  - "sftp.cybersoftbpo.com"
  - "CyberSoft partner SFTP"
  - "CyberSoft SFTP service"
tags: [sftp, openssh, aws, ec2, ebs, bindfs, cybersoft, chroot, security, ubuntu-20.04]
confidence: high
---

# CyberSoft partner SFTP service

Partner-facing SFTP endpoint for CyberSoft file exchange — public
hostname `sftp.cybersoftbpo.com`, TCP port 2233. Serves ~78 SFTP-only
user accounts across ~15 partner project namespaces, each chroot'd into
its own subtree of a shared ~600GB EBS data volume.

## Context

- **Owner:** CyberSoft (BPO / operations)
- **Environment:** production
- **Cloud:** AWS EC2 in us-west-2c
- **Instance:** single EC2 (currently `t2.small`; migration targets `t3a`
  or `t4g.medium` for 4GB RAM and unlimited-burst credit model)
- **Data:** ~600GB EBS `ext4` volume, bindfs FUSE for per-user ownership
  remap enabling OpenSSH `ChrootDirectory`

## Architecture

Standard OpenSSH-based SFTP with per-user chroot. Nothing exotic:

| Layer | Component |
|---|---|
| Protocol | OpenSSH `Subsystem sftp internal-sftp -f AUTH -l VERBOSE` |
| Auth | Password-only (via `/etc/shadow`); PAM enabled |
| Isolation | `ForceCommand internal-sftp` + per-user `ChrootDirectory` |
| Restrictions | `PermitTunnel no`, `AllowTcpForwarding no`, `AllowAgentForwarding no`, `X11Forwarding no` (per Match block) |
| Storage | EBS `ext4` mounted at `/mnt/sftp_root`, bindfs FUSE remap at `/mnt/sftp_root/bindfs` |
| Users | ~78 Linux users (`uid ≥ 1000`) organized by project prefix; no shell access (shell field empty or `/bin/sh`, overridden by `ForceCommand`) |
| Read-only variants | A subset use `ForceCommand internal-sftp -R` (upload-forbidden) |
| Cipher/MAC/KEX hardening | Drop-in at `/etc/ssh/sshd_config.d/00-hardening.conf` (deployed 2026-09-15) — modern AEAD ciphers only, ETM SHA-2 MACs, curve25519 + DH group16/18 KEX, `ssh-ed25519` + `rsa-sha2-*` host keys, `LogLevel VERBOSE` |
| Admin access | `ubuntu@` via SSH — should migrate to AWS SSM Session Manager |
| Monitoring | `telegraf` (InfluxDB metrics), `teleport` gateway |

### Why bindfs

OpenSSH's `ChrootDirectory` requires the chroot root to be owned by
`root` and not writable by the user. Partner users need write access
inside their chroot. Bindfs FUSE mounts a second view of the same tree
with dynamic ownership rewriting, satisfying both constraints
simultaneously. Elegant pattern, fstab-driven, no custom systemd unit:

```
bindfs#/mnt/sftp_root/bindfs /mnt/sftp_root/bindfs fuse create-with-perms=u+rw:g+rw 0 0
```

### Config shape

Currently ~550 lines of `sshd_config` with one `Match User` block per
SFTP user. Heavy duplication — every block repeats the same 6
restriction directives. Refactorable via a `Match Group sftpusers`
baseline + per-user `Match User` blocks that only vary
`ChrootDirectory` — ~85% smaller, semantically equivalent. The
refactor is on the migration checklist below.

## Known weaknesses (as of 2026-09-15)

- **Ubuntu 20.04.6 LTS** — end-of-standard-support 2025-04-30. Ubuntu
  Pro not attached; ~11 months of un-installed security patches. Last
  `apt` run was October 2024. Scanner flags **12 medium-severity CVSS
  3.0 findings** against OpenSSH `8.2p1-4ubuntu0.13`. About half are
  client-side (scanner flags `openssh-client` version); the rest split
  into "fixable by ESM backport" and "requires newer OpenSSH than 20.04
  ships."
- **Password auth only** — no pubkey. Credential rotation is manual; a
  leaked partner password has no revocation path short of `passwd -l`.
  Big posture gap, but constrained by "services must keep working with
  hardcoded credentials." Migration retains this model.
- **Instance undersized** — `t2.small` (2GB RAM) for 78 users.
  CPU-credit throttling risk under sustained transfer load.
- **Data volume 84% full** (471GB / 591GB usable). Capacity crunch
  approaching.
- **No `fail2ban` / brute-force protection.** With password auth, this
  is a gap.
- **`auditd` inactive.** No file-access audit trail beyond sshd session
  records.
- **`ufw` inactive.** Attack surface control lives entirely at the AWS
  Security Group layer.
- **Dead repo entries** — `bionic-*esm` in apt sources (leftover from
  an 18.04 → 20.04 upgrade). Cleanup on migration.
- **`teleport` pinned to v13** (multiple majors behind current).

## Cipher/MAC hardening (2026-09-15)

Drop-in `/etc/ssh/sshd_config.d/00-hardening.conf` removed:

| Category | Removed |
|---|---|
| Cipher | `aes128-cbc` |
| MAC | `hmac-sha1`, `hmac-sha1-etm@openssh.com` |
| KEX | `diffie-hellman-group1-sha1` |
| HostKey | bare `ssh-rsa` (SHA-1 signatures) |

The base `sshd_config` still contains the vestigial `Ciphers +aes128-cbc`
and `KexAlgorithms +diffie-hellman-group1-sha1` lines that originally
weakened the config for a legacy client — overridden by first-obtained-value
semantics from the drop-in (which loads earlier via `Include
/etc/ssh/sshd_config.d/*.conf`). Cleanup on migration.

## Migration plan (2026-09-15, not yet executed)

Fresh-instance cutover to Ubuntu **26.04 LTS** (fallback: 24.04 LTS),
retaining the same OpenSSH-based architecture. Explicit operator
constraint: automated service integrations must keep working with
existing credentials — no move to SFTPGo, no move to AWS Transfer
Family, no move to S3 storage.

### Key mechanic — EBS volume swap

Data volume detach from old instance, attach to new instance **in the
same AZ (us-west-2c)** — EBS volumes are AZ-scoped and cannot cross
AZ boundaries via attach. `/etc/fstab` mounts by UUID, so the mount
survives instance-type family change (Xen → Nitro; the OS device name
shifts from `/dev/xvdf` to `/dev/nvme1n1` but UUID mount is transparent
to that). Bindfs is also fstab-driven and portable. **No 471GB rsync
required.** SSH host keys are copied old → new so partners see the same
fingerprint (no "REMOTE HOST IDENTIFICATION HAS CHANGED" warnings).

### Rollback

- Before EIP move: reattach volume to old, mount, start sshd. <5 min.
- After EIP move: reverse EIP association, reattach volume the other
  way. <10 min. Old box stays warm 14 days precisely for this.

### Cutover downtime

Target: 30-60 min for the actual EBS swap + EIP move + verification.
Partner clients hit "connection refused" briefly, then reconnect
transparently once DNS resolves (EIP moves, hostname stays).

## Alternative architectures considered but declined (2026-09-15)

- **SFTPGo + S3 storage.** Purpose-built modern SFTP daemon, API-driven
  user management, S3-durable storage, first-class MFA, post-quantum KEX,
  built-in defender/fail2ban. Declined for now to preserve current
  service-integration model — automated partners with hardcoded
  credentials keep working unchanged. Strong candidate for a Q4/Q1
  project once the 26.04 migration stabilizes.
- **AWS Transfer Family.** Fully managed SFTP, no server maintenance,
  S3-backed. Declined on cost (~6-10× monthly infra cost) and chroot
  semantics change (S3 prefixes vs filesystem paths) — risk to
  automated integrations that hardcode paths.
- **In-place `do-release-upgrade` 20.04 → 22.04 → 24.04 → 26.04.**
  Three sequential LTS hops, each with its own failure surface;
  preserves accumulated snowflake debt; rollback requires snapshot
  restore. Declined in favor of fresh-instance cutover, which forces
  documenting the config and gives a DNS-flip rollback path.

## Followups (post-migration)

- Prune dead SFTP users (accounts with no `Match User` block).
- Introduce `fail2ban` for brute-force protection on password auth.
- Introduce `auditd` rules on `/mnt/sftp_root/bindfs` for
  compliance-grade file access audit trail.
- Enable CloudWatch Logs shipping for sshd session records.
- Migrate `ubuntu@` admin access to SSM Session Manager — remove admin
  SSH exposure to the internet entirely; the SFTP service (:2233) becomes
  the only public port.
- Gradual pubkey rollout for **human** users only (per-user
  `AuthenticationMethods publickey`), keeping password auth for service
  accounts.
- Capacity: either provision a larger EBS (1TB) during the migration
  swap, or schedule a growth cutover in 3-6 months.
- Revisit SFTPGo/S3 architecture as a separate project after 26.04
  stabilizes.

## Related

- [[wiki/organizations/cybersoft]] — parent organization (service catalog
  and other CyberSoft-owned systems)
- [[wiki/projects/aws-tf-network]] — the VPC this instance lives in
- [[wiki/projects/cybersoftbpo-website]] — sibling CyberSoft project
  (public website rebuild)
- [[wiki/solutions/vpn-fails-when-another-vpn-agent-installed]] — ZPA
  route hijack pattern first surfaced trying to reach this box
- [[wiki/runbooks/ops-communication-templates]] — maintenance
  announcement template used for the hardening + migration windows
