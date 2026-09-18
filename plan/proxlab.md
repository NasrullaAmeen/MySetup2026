---
tags: [plan, proxlab, proxmox, homelab, hardware]
created: 2026-09-17 15:59:00 +05
modified: 2026-09-18 09:53:00 +05
---

# ProxLab - server note (homelab)

Stationary homelab Proxmox host. Will be carved from the X99 workstation
(hostname `omarchy`, currently running Omarchy/Arch as a desktop). Independent
from ProxDev (this laptop). Plan docs for that host: [proxdev.md](proxdev.md),
[networking.md](networking.md).

## Identity

| | |
|---|---|
| Name | **ProxLab** (host) |
| Role | HomeLab Proxmox server |
| Current hostname | `omarchy` -> will become `proxlab` |
| Current OS | Omarchy (Arch), kernel 7.2.5-3-omarchy |
| TODO | Preserve `/etc/hostname`, PVE install (like ProxDev: `pve-no-subscription`, user instead of root, root SSH off) |

## Hardware stack (read from this PC, 2026-09-18)

### CPU

- **2x Intel Xeon E5-2660 v4** (Broadwell-EP), 14C/28T each = **28 cores / 56 threads**
- 2.0 GHz base / 3.2 GHz turbo, 35 MiB L3 per socket, TDP 105 W
- Flags: VT-x, **VT-d** (active - 92 IOMMU groups), AVX2, AES-NI
- One NUMA node exposed (0-55)

### Memory

- **109 GiB DDR4** seen by the OS (ECC RDIMM; 4ch per socket)
- Swap: zram 109.9 G

### Storage

| Device | Size | Model | Connection | Content |
|---|---|---|---|---|
| nvme0n1 | 465.8G | Lexar SSD NM710 500GB (Longsys, DRAM-less Gen4) | PCIe NVMe | **Omarchy now**: LUKS -> btrfs @, /boot 2G; future: PVE root+ISO candidate |
| sda | 931.5G | WDC WDS100T2B0B-00YS70 (WD Blue 3D, SATA SSD) | SATA | **VM disk pool candidate** |
| sdb | 931.5G | ST1000LM035-1RK172 (Seagate BarraCuda 2.5, SATA HDD) | SATA | VM disk / bulk storage candidate |
| sdc | 931.5G | ST1000LM049-2GH172 (Seagate BarraCuda 2.5, SATA HDD) | SATA | VM disk / bulk storage candidate |
| sdd | 931.5G | PNY 1TB SATA SSD | **USB SATA** | btrfs "Hdscythe" backup target (shared with ProxDev backups) |
| zram0 | 109.9G | - | - | swap |

- 3x SATA bays usable: WD SSD + 2x Seagate HDD = plenty for VM disks. No L2ARC/ZIL NVMes; keep ZFS pool on the SATA SSD or use LVM-thin on a mirrored pair.

### GPU

- **Gigabyte Radeon RX 5700 XT Gaming OC** (Navi 10, `1002:731f`), 8 GB GDDR6,
  current driver `amdgpu`. MTBF-strong AC card = ideal single-GPU passthrough
  candidate for a Linux desktop VM (no iGPU on this board - the dGPU is the
  only display output until a GPU is attached to a VM).

### PCIe switch + IOMMU (GPU passthrough)

| Group | Device |
|---|---|
| 82 | 02:00.0 Navi 10 XL Upstream Port (PCIe switch) |
| 83 | 03:00.0 Navi 10 XL Downstream Port |
| 84 | 04:00.0 Radeon RX 5700 XT (VGA) |
| 85 | 04:00.1 Navi 10 HDMI Audio |

- Switch and GPU/audio are in **separate IOMMU groups** - passthrough the whole
  downstream subtree (groups 83+84+85) so the GPU + HDMI audio move together.
- Remaining PCI: NVMe (01:00.0), RTL8168 Ethernet (07:00.0), BCM4360 Wi-Fi
  (06:00.0) are each alone.

### Networking

- **Ethernet**: Realtek RTL8111/8168/8411 1GbE (`07:00.0`, r8169) -> primary uplink.
- **Wi-Fi**: Broadcom BCM4360 802.11ac (`06:00.0`) - not a passthrough target.
- Plan MGMT: **10.10.30.1** on **10.10.30.0/24** - `https://10.10.30.1:8006`.
- Internal VM bridge: vmbr0 **10.30.0.0/24** - gw 10.30.0.1.
- Router 10.10.10.1 routes 10.10.10.0/24 <-> 10.10.30.0/24.

### Board / misc

- Board: X99 (dual-socket, C610/X99 chipset - 6+4 SATA, xHCI).
- USB: standard X99 ports; no USB4/Thunderbolt.
- Audio: on-board HD audio + HDMI audio via GPU.

## VMs

_TODO: list ProxLab VMs + internal 10.30.0.x addresses. Skyline: RX 5700 XT
desktop VM (PCIe passthrough group 83/84/85), NAS/LXC services._

## Firewall / VLAN (future, per-server)

- Own firewall rules and its own VLAN setup - **not shared** with ProxDev.
- Cross-network to ProxDev (10.10.10.x / 10.20.0.0/24) **allowed only via
  explicit rules** permitting specific inter-server traffic.

## Open questions

- VM inventory + IP plan on 10.30.0.0/24.
- ZFS or LVM-thin for the VM pool (3 SATA disks available; no mirrors yet).
- Confirm uplink/interface names on the host; BCM4360 as a backup uplink only.

## See also

- ProxDev: [proxdev.md](proxdev.md)
- Shared networking context: [networking.md](networking.md)