---
tags: [plan, index, proxdev, proxlab]
created: 2026-09-18 09:40:00 +05
modified: 2026-09-18 09:40:00 +05
---

# Plan Index

Planning docs for the Proxmox setup (merged 2026-09-17 from MySetup).

- **[hardware.md](hardware.md)** - Full spec of the PHN16S-71 laptop (Acer
  Predator): CPU, memory, storage, GPUs, networking, USB/Thunderbolt, IOMMU
  groups, and thermals; how the constraints shape the ProxDev plan.
- **[proxdev.md](proxdev.md)** - The ProxDev build plan: Proxmox VE on this
  laptop hosting a multiverse of VMs (Main/Game/Dev tiers), GPU passthrough
  rules, storage layout, resource budget, and the install runbook. **IN USE** -
  running at 10.10.10.10:8006.
- **[networking.md](networking.md)** - Portable headless networking runbook:
  bridge/NAT design, multi-SSID Wi-Fi, routing, remote management,
  troubleshooting. Home LAN set at 10.10.10.10 (10.10.10.0/24).
- **[proxlab.md](proxlab.md)** - Notes for "ProxLab", a separate stationary
  homelab Proxmox server, independent from ProxDev.

## Layout

Two Proxmox hosts, each with its own private VM bridge and firewall/VLAN
policy, cross-reachable only through explicit rules:

| Host | Role | Home LAN MGMT | Internal VM bridge |
|---|---|---|---|
| **ProxDev** | PHN16S-71 laptop, in the homelab | 10.10.10.10 (10.10.10.0/24) | 10.20.0.0/24 |
| **ProxLab** | Stationary homelab server (future) | 10.10.30.1 (10.10.30.0/24) | 10.30.0.0/24 |

Live server notes: see [../docs/proxdev-notes.md](../docs/proxdev-notes.md)
and [../docs/README.md](../docs/README.md).