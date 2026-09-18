---
tags: [plan, proxlab, proxmox, homelab]
created: 2026-09-17 15:59:00 +05
modified: 2026-09-18 09:40:00 +05
---

# ProxLab - server note (homelab)

Separate Proxmox VE host - the stationary homelab server. Independent from
ProxDev (this laptop). Plan docs for that host: [proxdev.md](proxdev.md),
[networking.md](networking.md).

## Identity

| | |
|---|---|
| Name | **ProxLab** (host) |
| Role | HomeLab Proxmox server |
| Hostname | `proxlab` (TODO: confirm) |

## Network

| | |
|---|---|
| Home LAN (MGMT) | **10.10.30.1** on **10.10.30.0/24** - `https://10.10.30.1:8006` |
| Internal VM bridge | vmbr0 **10.30.0.0/24** - gw 10.30.0.1 |
| Router | 10.10.10.1 routes 10.10.10.0/24 <-> 10.10.30.0/24 |

## VMs

_TODO: list ProxLab VMs + internal 10.30.0.x addresses._

## Firewall / VLAN (future, per-server)

- Own firewall rules and its own VLAN setup - **not shared** with ProxDev.
- Cross-network to ProxDev (10.10.10.x / 10.20.0.0/24) **allowed only via
  explicit rules** permitting specific inter-server traffic.

## Open questions

- Hardware spec (CPU / RAM / storage / GPUs / IOMMU).
- VM inventory + IP plan on 10.30.0.0/24.
- Uplink/interface names on the host.

## See also

- ProxDev: [proxdev.md](proxdev.md)
- Shared networking context: [networking.md](networking.md)