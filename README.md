# MySetup 2026

Planning docs for my personal hardware and homelab/Proxmox setup.

## Contents

- **[HARDWARE.md](HARDWARE.md)** — Full spec of the PHN16S-71 laptop (Acer Predator): CPU, memory, storage, GPUs, networking, USB/Thunderbolt, IOMMU groups, and thermals. Also covers how these constraints shape the ProxDev plan.
- **[NETWORKING.md](NETWORKING.md)** — Portable headless networking runbook for running Proxmox on a laptop with no permanent uplink: bridge/NAT design, multi-SSID Wi-Fi, routing, remote management, and troubleshooting.
- **[ProxDev.md](ProxDev.md)** — The "ProxDev" build plan: Proxmox VE on the laptop hosting a multiverse of VMs (Main/Game/Dev tiers), GPU passthrough rules, storage layout, resource budget, and the full install runbook.
- **[ProxLab.md](ProxLab.md)** — Notes for "ProxLab", a separate stationary homelab Proxmox server, independent from ProxDev.

## Layout

Two Proxmox hosts, each with its own private VM bridge and firewall/VLAN policy, cross-reachable only through explicit rules:

| Host | Role | Home LAN | Internal VM bridge |
|---|---|---|---|
| **ProxDev** | Laptop (PHN16S-71), portable | 10.10.20.1/24 | 10.20.0.0/24 |
| **ProxLab** | Stationary homelab server | 10.10.30.1/24 | 10.30.0.0/24 |
