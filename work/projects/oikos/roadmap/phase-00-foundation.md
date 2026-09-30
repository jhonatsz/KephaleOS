---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-0, foundation, proxmox, ccna]
confidence: high
---

# Phase 0 — Foundation

## Objective

Get a documentation system, an address plan, and a working Proxmox host on the bench. Zero changes to the household network. When Phase 0 ends: I can spin up VMs, I know exactly what I'm building, and every subsequent phase has a place to land.

## Why now

Everything downstream needs (a) a place to record decisions and current-state, and (b) a compute substrate to host VMs. Without both, Phase 1 (OPNsense) has no landing surface and Phase 3 (DNS/DHCP) has nowhere to run.

## Prerequisites

None — this is the foundation.

## Namespace decision (resolved)

**Decided 2026-09-18**: internal zone is `oikos.home.arpa` (RFC 8375-compliant subdomain of `home.arpa`). See [[../../../../wiki/decisions/oikos-0001-dns-namespace|ADR oikos-0001]].

If the T14 was momentarily installed as `pve-t14-01.homelab.arpa`, apply the correction now (Phase 0 is the last cheap moment). As root on the T14:

```bash
hostnamectl set-hostname pve-t14-01.oikos.home.arpa

# edit /etc/hosts so the 127.0.1.1 line reads:
# 127.0.1.1  pve-t14-01.oikos.home.arpa pve-t14-01

pvecm updatecerts --force
systemctl restart pveproxy pvedaemon

hostname -f   # verify → pve-t14-01.oikos.home.arpa
```

## Hardware

| Item | Status | Required? |
|---|---|---|
| Lenovo ThinkPad T14 Gen 2 (40 GB · 4C/8T · 512 GB SSD) | OWNED | Required |
| USB Ethernet adapter | not yet | Optional (helps in Phase 1 rehearsal for dual-NIC OPNsense) |
| USB 3 stick (≥ 8 GB) for installer | assumed on hand | Required |

**Buy trigger**: none. Reuse only.

## Software

- **Proxmox VE** — latest stable (8.x line as of 2026-09)
- **Debian netinst rescue** (optional, only if Proxmox installer misbehaves)

## Network changes

None on the household. The T14 sits on the ISP router's DHCP during Phase 0. IP transition to `172.27.10.11/24` happens as part of Phase 2 (VLAN rollout).

## Implementation sequence

1. **Verify hardware** — BIOS check: virtualization (VT-x) enabled, iommu on if you plan to pass through USB/PCIe later.
2. **Download Proxmox ISO**; verify SHA256 against Proxmox's published checksum.
3. **Create USB installer** (`dd` on macOS/Linux; Rufus on Windows).
4. **Install with the documented parameters** — see [[../nodes/pve-t14-01]]:
   - Filesystem: `ext4`
   - Volume manager: LVM + LVM-thin
   - `hdsize` ~476 GiB, `swap` 8 GiB, `root` 64 GiB, `minfree` 16 GiB, `maxvz` automatic
5. **Set hostname correctly** — resolve the namespace decision above before this step, then set once.
6. **Post-install hygiene** (as root):
   - Disable enterprise repo (`/etc/apt/sources.list.d/pve-enterprise.list`)
   - Add community repo (`pve-no-subscription`)
   - `apt update && apt full-upgrade`
   - Reboot into new kernel
   - Suppress the subscription-nag popup (optional — script exists; document in the runbook)
7. **SSH hardening**:
   - Push your workstation's SSH key: `ssh-copy-id root@<T14-DHCP-IP>`
   - Set `PermitRootLogin prohibit-password` (keys only) and `PasswordAuthentication no` in `/etc/ssh/sshd_config`
   - `systemctl restart sshd`
   - Verify from a **second** terminal before closing the first (safety net)
8. **Root password**: rotate; store only in your password manager. Never in git.
9. **First VM template** — Debian cloud image:
   - `wget` current Debian 12 generic-cloud qcow2
   - Import to Proxmox with `qm importdisk`
   - Configure `qm set` with cloud-init, serial console, and 2 vCPU / 2 GB baseline
   - Convert VM to template
10. **Clone the template**; run `apt update` inside; confirm it gets DHCP from the ISP router.
11. **Snapshot the Proxmox config** — commit `/etc/pve/` structure summary (no keys!) and `pveversion -v` output to `~/workspace/personal/homelab/nodes/pve-t14-01/`.
12. **Write the runbook** — `../runbooks/proxmox-install.md` from the actual experience, not from memory. Every deviation from these steps gets recorded.

## CCNA topics reinforced

- OSI / TCP-IP layer model — Proxmox networking uses Linux bridges (L2) with IP on top (L3); explicit muscle for "which layer is this happening on?"
- IPv4 addressing basics — subnet of the ISP router, first-hop MAC/IP interaction
- ARP — watch `ip neigh` populate as the T14 talks to the gateway
- DHCP client behavior — Discover/Offer/Request/Ack visible in `journalctl`
- `/etc/hosts` vs. DNS — order of resolution, `nsswitch.conf`
- Static vs. dynamic addressing — you'll do both before this phase ends

## Packet Tracer / EVE-NG lab

N/A directly for Phase 0 — but this is when the **[[../study/packet-tracer|Packet Tracer study track]] kicks off in parallel**. Install Packet Tracer and start Lab 01 anytime during Phase 0; it doesn't gate anything but should be running by the time you enter Phase 1.

## Physical Oikos lab exercise

1. From your workstation, SSH into `pve-t14-01` using its DHCP address.
2. On the Proxmox host: `ip addr`, `ip route`, `ip neigh` — describe out loud what each line means.
3. Ping the ISP router gateway. Ping `1.1.1.1`. Ping `google.com`. Explain which of L3/DNS each ping proves.
4. Boot a Debian VM from the template. From that VM, `traceroute 8.8.8.8` — count the hops, identify NAT boundary.
5. On the Proxmox host: `bridge link show`, `brctl show` — the VM's veth attaches to `vmbr0`. Draw this on paper.

## Break/fix drill

**Set a timer.** Then:

- On the Proxmox host, misconfigure `/etc/network/interfaces` — remove the address line from `vmbr0`. Do not commit `ifreload`. Instead: **reboot** the box. SSH is now dead.
- Recover from the physical T14 keyboard: log in as root, restore the file from `.orig`, `ifup vmbr0`, verify SSH returns.
- Write down: what did you see on the console? Which commands worked without networking? Which didn't?

This drill is the "the network went away" muscle memory — you will do this for real one day.

## Verification commands

Host:

```bash
hostnamectl status
ip addr
ip route
ip neigh
ping -c 3 <isp-gateway>
ping -c 3 1.1.1.1
ping -c 3 google.com
resolvectl status  # or: cat /etc/resolv.conf
pvesh get /nodes
qm list
pveversion -v
```

Inside VM:

```bash
ip addr
ip route
ping -c 3 <proxmox-host>
ping -c 3 1.1.1.1
traceroute 1.1.1.1
```

## Security posture at this phase

- SSH: **keys only**, password auth disabled
- Proxmox web UI: strong root password (in password manager only), UI reachable only on LAN
- **No inbound from Internet** — do NOT port-forward to the T14
- Firewall = ISP router (weak; acceptable only because nothing sensitive lives here yet). Phase 1 replaces it.
- No secrets in git — `.gitignore` already covers it; verify with `git check-ignore` before committing anything

## Monitoring coverage

- No stack yet (Phase 5 starts real observability). For Phase 0, spot-check:
  - `pveperf` — baseline of I/O
  - `df -h`, `free -h`, `uptime`
  - `journalctl -p err -b`

## Kephaleos docs to create / update

- **Create**: `work/projects/oikos/runbooks/proxmox-install.md` (write from real experience, not from memory)
- **Create**: `work/projects/oikos/adr/0001-dns-namespace.md` (record the namespace decision)
- **Update**: `work/projects/oikos/current-state.md` — T14 status DEPLOYED once Proxmox is running
- **Update**: `work/projects/oikos/nodes/pve-t14-01.md` — real install parameters used, DHCP address, deviations from plan
- **Append**: `work/projects/oikos/CHANGELOG.md`
- **Later** (post-phase): promote `runbooks/proxmox-install.md` → `wiki/runbooks/proxmox-install.md`

## Diagram updates

None required — no architecture change on the wire. Add a Mermaid "current state" mini-diagram to `current-state.md` if it helps: T14 as a single box hanging off the ISP router.

## Completion criteria (do not advance until all are true)

- [ ] Namespace decision recorded in `../adr/0001-dns-namespace.md`
- [ ] Hostname on the T14 matches the decision, and matches every doc
- [ ] Proxmox installed with documented parameters; `pveversion` recorded
- [ ] Community repo enabled; all packages updated
- [ ] SSH keys-only; password auth disabled; verified from a second terminal
- [ ] Root password rotated and stored in password manager only
- [ ] Debian cloud template imported, one VM cloned and reachable
- [ ] Break/fix drill completed and notes captured
- [ ] `runbooks/proxmox-install.md` written from real experience
- [ ] `current-state.md`, `nodes/pve-t14-01.md`, `CHANGELOG.md` updated
- [ ] Config snapshot committed to `~/workspace/personal/homelab/` (sanitized — no keys)

Only when every box is ticked: move to Phase 1. (The Packet Tracer study track runs in parallel and doesn't gate anything.)
