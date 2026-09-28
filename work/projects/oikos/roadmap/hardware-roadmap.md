---
type: project
status: active
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, hardware, purchasing, cost]
confidence: high
---

# Oikos — Consolidated Hardware & Materials Roadmap

Single-page purchase view of every hardware item Oikos needs, when to buy it, and roughly what it costs. Prices are ballpark 2026 US-market — adjust for your region and used-market availability.

**Buy-timing convention:**

- **BUY NOW** — needed for the current or next phase
- **BUY SOON** — 1–2 phases out
- **WAIT** — later; do not pre-buy
- **OWNED** — already on hand
- **PROPOSED** — under evaluation, no commitment

## Master table (grouped by phase)

| Phase | Item | Purpose | Req? | Timing | Rough cost |
|---|---|---|---|---|---|
| 0 | Lenovo ThinkPad T14 Gen 2 (40 GB · 4C/8T · 512 GB) | Bootstrap Proxmox | Req | OWNED | — |
| 0 | USB installer (≥ 8 GB) | Proxmox install media | Req | OWNED | — |
| 1 | Firewall appliance (Protectli VP2410/VP2420, Qotom Q35xxG4, CWWK N100 dual/quad NIC) | OPNsense host | Req | BUY NOW | $300–500 |
| 1 | USB-serial console cable | Recovery / initial config | Rec | BUY NOW | $15 |
| 1 | Cat6 patch cables (×2) | WAN + LAN | Req | already own | — |
| 2 | Managed L2+/L3 switch, 24-port, 802.1Q, PoE (≥ 8 PoE ports) — MikroTik CRS328-24P-4S+RM / Ubiquiti USW-Pro-24-PoE / Cisco Catalyst 1200/1300 / TP-Link TL-SG3428MP | Core switching, VLANs, PoE | Req | BUY NOW | $300–700 |
| 2 | Cat6 patch cables (×6) | Rack + trunks | Req | BUY NOW | $25 |
| 2 | 1U rack shelf | Switch mounting | Opt | BUY NOW | $25 |
| 3 | — | Runs as VMs | — | — | — |
| 3 | UPS (APC Back-UPS Pro 1500 / CyberPower CP1500PFCLCD) | Firewall+switch+Proxmox power protection | Rec | BUY SOON | $150–250 |
| 4 | Wi-Fi 6/6E APs, PoE, VLAN-aware (~3–5 units) — Ubiquiti U6-Lite/U6-Pro/U6-LR / TP-Link Omada EAP670 / Aruba Instant On AP22 | Household Wi-Fi across 5 floors | Req | BUY NOW | $100–250 each × 3–5 |
| 4 | Cat6 structured drops + keystones + faceplates (305 m box + accessories) | Cable each AP location | Req | BUY NOW | $100–200 + labor |
| 4 | 24-port Cat6 patch panel | Terminate structured cabling | Rec | BUY NOW | $40–80 |
| 4 | Wall rack / open frame (6U–12U) | Central rack | Rec | BUY NOW | $100–200 |
| 4 | Floor switch(es), 8-port L2 PoE — Ubiquiti USW-Flex-Mini / TP-Link TL-SG108PE | Injectors where structured Cat6 impractical | Opt | conditional | $80–150 |
| 4 | Cable tester + toner (Klein VDV501 / Fluke MicroScanner) | Verify drops | Rec | BUY NOW | $40–200 |
| 4 | Punch-down tool + crimper | Terminate keystones | Req | BUY NOW | $30 |
| 4 | Fish tape + drill + long bits | Cabling | Req for cabling | BUY NOW | varies |
| 5 | UPS (if not bought earlier) | Power protection extended | Rec | BUY NOW | $150–250 |
| 6 | Additional RAM for T14 (upgrade to max) | Nested EVE-NG headroom | Cond | BUY as needed | $80–150 |
| 7 | NAS chassis, 4–6 bay — Synology DS923+/DS1522+ / TrueNAS Mini X / custom (Fractal Node 304 + N100/J5040 board) | Storage host | Req | BUY NOW | $400–1,200 |
| 7 | NAS drives, 4× 4–8 TB CMR NAS-grade — WD Red Plus / Seagate IronWolf / Toshiba N300 | Storage capacity | Req | BUY NOW | $100–200 each |
| 7 | Cold-spare drive (1×) | Faster resilver on failure | Rec | BUY NOW | 1× drive cost |
| 7 | NVMe cache | Metadata / small-IO speedup | Opt | BUY WAIT | $60–120 |
| 7 | Off-site backup drive (USB 8+ TB) *or* Backblaze B2 plan | 3-2-1 backup | Req | BUY NOW | $120 + power *or* $6/TB/month |
| 8 | Second Proxmox host, 32+ GB RAM, 8+ cores — Lenovo ThinkStation P520 (used) / M720q-M920q family / custom AM4-AM5 build / Dell OptiPlex Micro | HA cluster, DC02 host | Req | BUY NOW | $400–1,200 |
| 8 | Second NIC for corosync separation | Cluster latency stability | Rec | BUY NOW | $20 |
| 9 | — | VMs on existing cluster | — | — | — |
| 10 | GPU, 24 GB VRAM — RTX 3090 (used) / RTX 4090 / RTX A5000 (used) | LLM inference | Req | BUY NOW | $700–1,600 |
| 10 | AI host chassis, 750 W+ PSU, 16+ cores, 64+ GB RAM, PCIe passthrough — ThinkStation P520 (used, base) / custom AM5 Ryzen 9 / Dell Precision 5820 | GPU host | Req | BUY NOW | $500–1,500 (base, add GPU) |
| 10 | NVMe (2 TB) for models | Model hot storage | Rec | BUY NOW | $130–250 |
| 10 | UPS uprated for GPU draw (1500 VA min line-interactive) | Sustained-load power protection | Cond | BUY as needed | $200 |
| 11 | — | Runs on existing compute | — | — | — |

## Rollup by cost band

| Band | Phases | Includes |
|---|---|---|
| Sub-$500 (starter) | 0–1 | Firewall appliance + cables |
| $500–$1,500 (network core) | 1–4 | Firewall + switch + APs + cabling + UPS |
| $1,500–$3,500 (compute + storage) | 5–8 | UPS, NAS + drives, second Proxmox host, cluster NIC |
| $2,500–$4,000 (AI stack) | 10 | GPU + host + NVMe + UPS uprate |

Numbers assume mostly used enterprise gear + a few new items where used doesn't make sense (drives, cables, PSU-in-warranty).

## Optional / deferred / proposed

- **Second NAS or offsite server** at a friend's / relative's house for real geographic redundancy — WAIT until Phase 7 backup workflow proves cloud-only isn't enough
- **KVM-over-IP** (PiKVM or Lantronix Spider) — quality-of-life for headless recovery; PROPOSED after Phase 8
- **10 GbE upgrade** — PROPOSED after Phase 7 if NAS→cluster throughput becomes the bottleneck
- **Second AP model for coverage edge cases** — outdoor unit if outdoor coverage needed later
- **Rack PDU (metered/switched)** — PROPOSED when the rack has 4+ devices worth remote power-cycling
- **Managed L3 core (upgrade from Phase 2 switch)** — WAIT unless you outgrow the 24-port unit or want stack/HA
- **Dedicated logging drive on NAS** — PROPOSED if Loki retention grows beyond expected

## Cost-avoidance principles

1. **Used enterprise > new consumer** for anything not touching drives/PSUs (chassis, mobos, switches with recent firmware).
2. **CMR only** for NAS drives — SMR corrupts ZFS resilvers.
3. **PSU warranty matters** — buy new for anything that dies loudly.
4. **Buy the cheapest thing that meets the phase's completion criteria.** Do not buy for the phase after next.
5. **Standardize on one AP vendor** — cross-vendor controller pain isn't worth the savings.
6. **Skip 10 GbE** unless a specific workload demands it (measured, not imagined).
7. **Don't buy a rack you'll outgrow in a year** — but do buy one bigger than you think you need *today*.

## Related

- [[phase-plan]] — the phase sequence this maps to
- [[../inventory/nodes]] — who owns what today
- [[../current-state]] — what is DEPLOYED right now
- Individual phase pages for detailed hardware rationale
