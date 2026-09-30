---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-4, wifi, wireless, multi-floor, poe]
confidence: high
---

# Phase 4 — Wi-Fi + Multi-Floor Distribution

## Objective

Reliable wired Ethernet backhaul and Wi-Fi coverage across the **5 floors** of the house. Multiple SSIDs mapped to VLANs (home, iot, guest, camera-mgmt).

## Why now

VLANs exist (Phase 2), DNS/DHCP exist (Phase 3). Wi-Fi expansion needs both. Doing it earlier means single-SSID chaos and painful re-cabling.

## Prerequisites

- Phase 2 complete (managed switch, VLANs)
- Phase 3 complete (DHCP/DNS per VLAN)
- A rough floor plan with device counts per floor

## Hardware

| Item | Required? | Buy timing | Candidates | Rough cost |
|---|---|---|---|---|
| **Access points**, Wi-Fi 6 or 6E, PoE, VLAN-aware, ceiling-mount preferred | **Required** (~3–5 units for 5 floors) | BUY NOW | Ubiquiti U6-Lite / U6-Pro / U6-LR · TP-Link Omada EAP670 · Aruba Instant On AP22/AP25 | US$100–250 each |
| **Structured Cat6 drops** — one to each AP location | **Required** | BUY NOW | 305m box of Cat6 solid + keystones + faceplates | US$100–200 + labor |
| Cat6 patch panel (24-port) | Recommended | BUY NOW | any brand | US$40–80 |
| Small wall rack or open frame | Recommended | BUY NOW | 6U-12U | US$100–200 |
| Additional floor switch(es) if PoE-injecting cable runs is impractical | Optional | conditional | small 8-port L2 PoE (Ubiquiti USW-Flex-Mini, TP-Link TL-SG108PE) | US$80–150 |
| Cable tester + toner | Recommended | BUY NOW | Klein Tools VDV501 · Fluke MicroScanner | US$40–200 |
| Punch-down tool + crimper | Required | BUY NOW | any brand | US$30 |
| Fish tape / drill / long bits | Required for cabling | BUY NOW | any | varies |

**Note**: Wi-Fi controller software is usually free/bundled (UniFi Network Controller, Omada Controller). Run it as a VM on Proxmox VLAN 20 — hostname e.g. `wifi01.oikos.home.arpa`.

## Software

- Vendor controller (UniFi / Omada / etc.) as a VM
- Firmware updates on APs

## Network changes

- Cat6 drops from central rack to each AP location
- Managed switch PoE ports = one per AP
- APs on VLAN 10 (mgmt) for controller communication
- SSID → VLAN mapping:
  - `Oikos-Home` (WPA3-Personal) → VLAN 100
  - `Oikos-IoT` (WPA2-Personal — many IoT devices don't support WPA3) → VLAN 60
  - `Oikos-Guest` (Guest network with captive portal optional) → VLAN 80
  - Optional `Oikos-Cam-Setup` (temporary during camera onboarding) → VLAN 70

## Implementation sequence

1. **Walk the house** — mark AP candidate locations on the floor plan (target: minimize walls between AP and each usage area; central ceiling > corner shelf).
2. **Wi-Fi site survey** — free tools: WiFi Analyzer (Android), NetSpot (macOS free tier). Note current SSIDs, channels, RSSI.
3. **Order hardware** (see table above). Coordinate cabling day with family.
4. **Cabling day**:
   - Run structured Cat6 from central rack to each AP location; terminate to keystones + patch panel
   - Test every drop with cable tester (wire map + length)
   - Label both ends
5. **Install controller** — VM on Proxmox VLAN 20, adopt APs, apply default AP config
6. **Configure SSIDs**:
   - VLAN tagging per SSID
   - Band steering off initially (troubleshooting first, optimization later)
   - Minimum RSSI enabled
   - Fast roaming (802.11r/k/v) on
7. **Test roaming** — walk between floors with a phone playing a video/call, watch handoff (`iw dev wlan0 link` on Linux)
8. **Tune channels** — set 2.4 GHz to non-overlapping (1, 6, 11); 5 GHz DFS if allowed in region
9. **PoE budget check** — total AP power draw ≤ switch PoE budget
10. **Old ISP router**: fully retired now if not already
11. **Document** — floor plan with AP positions, PoE budget, channel plan

## CCNA topics reinforced

- Wireless 802.11 fundamentals (a/b/g/n/ac/ax, bands, channels)
- SSID → VLAN mapping (translational bridging concept)
- PoE / PoE+ / PoE++ classes
- Channel plans and DFS
- Wireless authentication (WPA2 vs WPA3)
- Fast roaming (802.11r/k/v)

## Study track lab

Packet Tracer has limited wireless realism. This phase's study is more site-survey + reading than lab. Optional: EVE-NG doesn't help here either — Phase 4 is a physical-work phase.

## Physical Oikos lab exercise

- With WiFi Analyzer, walk each floor; record RSSI at every corner
- On a laptop: `nmcli device wifi list` — identify each AP by BSSID
- Test roaming: start a `ping -i 0.2 <gateway>` and walk between floors — count dropped packets during handoff
- On the switch: `show power inline` — actual PoE draw per port

## Break/fix drill

1. Unplug the AP that carries the SSID you're on. Observe handoff time to nearest AP.
2. Change one AP's channel to overlap with another (both channel 6 on 2.4 GHz). Observe throughput degradation via speedtest.
3. Fix.

## Verification commands

Switch:

```
show power inline
show interfaces status
show mac address-table | include <AP-MAC>
```

Controller (SSH into AP):

```bash
iwconfig
iwlist scan | head -60
info                          # UniFi CLI
```

Client:

```bash
iwconfig
iw dev wlan0 link
ping -c 100 -i 0.2 <gateway>  # roaming loss test
```

## Security posture at this phase

- Guest SSID: rate-limited, isolation on (no client-to-client), captive portal optional
- IoT SSID: no inter-VLAN reach; deny inbound from home
- WPA3-Transition only if you have very old devices; else WPA3-only
- Disable WPS everywhere
- AP admin UI: reachable only from mgmt VLAN

## Monitoring coverage

- Controller dashboards (client counts, retry %, throughput)
- SNMP from APs to Prometheus (if Phase 5 already done — otherwise defer)

## Kephaleos docs to create / update

- **Create**: `network/wifi-site-survey.md` (floor plan, RSSI, decisions)
- **Create**: `runbooks/ap-adoption.md`
- **Create**: `runbooks/ssid-vlan-mapping.md`
- **Create**: `inventory/cabling.md` (drop inventory)
- **Update**: `current-state.md`, `inventory/nodes.md`, `architecture/address-plan.md` (AP mgmt IPs)
- **Append**: `CHANGELOG.md`
- **New ADR**: `adr/000N-wifi-vendor.md`

## Diagram updates

- Physical multi-floor diagram (new): rack → per-floor drops → APs
- Update master diagram: APs on the tree

## Completion checklist

- [ ] All AP drops terminated + tested
- [ ] All APs powered via PoE, adopted, on latest firmware
- [ ] SSIDs live and mapped to correct VLANs
- [ ] Coverage verified on every floor (RSSI ≥ −70 dBm in each usage area)
- [ ] Roaming tested and acceptable (< 200 ms handoff)
- [ ] Guest network isolated and rate-limited
- [ ] PoE budget within switch capacity
- [ ] Docs + diagrams + inventory updated
