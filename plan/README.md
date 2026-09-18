---
tags: [plan, index, omalaptop, proxlab]
created: 2026-09-18 09:40:00 +05
modified: 2026-09-18 18:25:03 +05
---

# Plan Index

Planning docs for the Proxmox setup (merged 2026-09-17 from MySetup).

- **[hardware.md](hardware.md)** - Full spec of the PHN16S-71 laptop (Acer
  Predator): CPU, memory, storage, GPUs, networking, USB/Thunderbolt, IOMMU
  groups, and thermals; how the constraints shape the OmaLaptop plan.
- **[omalaptop.md](omalaptop.md)** - The OmaLaptop build plan: Omarchy
  desktop + nested **ProxDev** PVE VM hosting the lab tier (Main/Game/Dev
  tiers), GPU passthrough rules, storage layout, resource budget, install
  runbook. PIVOTED 2026-09-18 (no bare-metal PVE).
- **[networking.md](networking.md)** - Portable headless networking runbook:
  bridge/NAT design, multi-SSID Wi-Fi, routing, remote management,
  troubleshooting. Home LAN set at 10.10.10.10 (10.10.10.0/24).
- **[proxlab.md](proxlab.md)** - Notes for "ProxLab", a separate stationary
  homelab Proxmox server, independent from OmaLaptop. Hardware captured: X99
  dual-Xeon E5-2660 v4 workstation (omarchy) - 28C/56T, 109G RAM, RX 5700 XT.

## Layout

Two Proxmox hosts, each with its own private VM bridge and firewall/VLAN
policy, cross-reachable only through explicit rules:

| Host | Role | Home LAN MGMT | Internal VM bridge |
|---|---|---|---|
| **OmaLaptop** | PHN16S-71 laptop, Omarchy host; nested ProxDev PVE VM | 10.10.10.10 (10.10.10.0/24) | 10.20.0.0/24 |
| **ProxLab** | Stationary homelab server (future) | 10.10.30.1 (10.10.30.0/24) | 10.30.0.0/24 |

Live server notes: see [../docs/omalaptop-notes.md](../docs/omalaptop-notes.md)
and [../docs/README.md](../docs/README.md).