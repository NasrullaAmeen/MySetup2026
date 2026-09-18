# Setup 2026

Personal notes + planning docs for the Proxmox/homelab setup, running on the
Acer Predator PHN16S-71 laptop (host **ProxDev**, PVE node `prodev`).

- **[plan/](plan/)**: Plan docs - hardware, ProxDev build, networking,
  ProxLab.
- **[docs/](docs/)**: Live notes - access/credentials, hardware status,
  tasks, changelog. See [docs/README.md](docs/README.md) for the index.
- **[research/](research/)**: Research notes (API, storage, networking,
  security, AI stack, monitoring, ...). See [research/README.md](research/README.md).

## Contents (plan/)

- **[plan/hardware.md](plan/hardware.md)** - Full spec of the PHN16S-71
  laptop (Acer Predator): CPU, memory, storage, GPUs, networking,
  USB/Thunderbolt, IOMMU groups, thermals; how the constraints shape the
  ProxDev plan.
- **[plan/proxdev.md](plan/proxdev.md)** - The ProxDev build plan: Proxmox VE
  on this laptop hosting a multiverse of VMs (Main/Game/Dev tiers), GPU
  passthrough rules, storage layout, resource budget, install runbook.
  **IN USE** at https://10.10.10.10:8006.
- **[plan/networking.md](plan/networking.md)** - Portable headless networking
  runbook: bridge/NAT design, multi-SSID Wi-Fi, routing, remote management,
  troubleshooting.
- **[plan/proxlab.md](plan/proxlab.md)** - Notes for "ProxLab", a separate
  stationary homelab Proxmox server, independent from ProxDev.

## Agent/doc conventions

Read [AGENTS.md](AGENTS.md) first - it defines how AI agents operate in this
repo (frontmatter, ASCII-only notes, docs update rules). Also see
[CLAUDE.md](CLAUDE.md).

## Layout

Two Proxmox hosts, each with its own private VM bridge and firewall/VLAN
policy, cross-reachable only through explicit rules:

| Host | Role | Home LAN MGMT | Internal VM bridge |
|---|---|---|---|
| **ProxDev** | PHN16S-71 laptop, in the homelab | 10.10.10.10 (10.10.10.0/24) | 10.20.0.0/24 |
| **ProxLab** | X99 dual-Xeon workstation (omarchy), station | 10.10.30.1 (10.10.30.0/24) | 10.30.0.0/24 |