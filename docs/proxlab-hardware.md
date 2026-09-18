---
tags: [proxmox, hardware, server, inventory, proxlab]
created: 2026-09-18 10:30:00 +05
modified: 2026-09-18 18:25:03 +05
---

# ProxLab - Full Hardware Stack

## Summary

| Component | Model / Detail |
|-----------|----------------|
| CPU | 2x Intel Xeon E5-2660 v4 (Broadwell-EP) - 28 cores / 56 threads |
| RAM | 109 GiB DDR4 ECC RDIMM (8 channels, dual socket) |
| GPU | Gigabyte Radeon RX 5700 XT Gaming OC (Navi 10, 8 GB GDDR6) |
| NVMe | Lexar SSD NM710 500 GB (Longsys, DRAM-less) |
| SATA | WD Blue 1 TB SSD + 2x Seagate 1 TB HDD (5400/7200 rpm) |
| USB | PNY 1 TB SATA SSD ("Hdscythe" backup, detached between runs) |
| NIC | Realtek RTL8168 1GbE + Broadcom BCM4360 802.11ac |
| Board | X99 dual-socket (C610/X99 chipset, no BMC/IPMI) |

## Platform

- **Node**: ProxLab (planned 10.10.30.1) - **not yet installed**
- **Current OS**: Omarchy (Arch), kernel 7.2.5-3-omarchy, hostname `omarchy`
- **Target**: Proxmox VE on nvme0n1 (PVE 9.x, like OmaLaptop)
- **Boot**: UEFI, Secure Boot status TBD on PVE install
- **IOMMU**: VT-d active - 92 IOMMU groups measured

## CPU

- **Model**: 2x Intel Xeon E5-2660 v4
- **Sockets**: 2, **Cores**: 28, **Threads**: 56 (14C/28T per socket)
- **Clock**: 2.0 GHz base / 3.2 GHz turbo, TDP 105 W each
- **Cache**: 35 MiB L3 per socket
- Notes: VT-x, VT-d, AVX2, AES-NI; **no AVX-512**; **no iGPU** on board

## Memory

- **Total RAM**: 109 GiB DDR4 ECC RDIMM
- **Channels**: 4 per socket (8 total), ECC SEC/DED supported
- **Swap**: zram 109.9 G
- Note: ECC reporting to verify with `rasdaemon`/`edac-util` after PVE install

## GPU

- **GPU**: Gigabyte Radeon RX 5700 XT Gaming OC - `1002:731f`, Navi 10, 8 GB GDDR6
- **Audio**: Navi 10 HDMI Audio - `1002:ab38`
- **Driver**: amdgpu (current); for VM passthrough use vfio-pci (see research/25)
- Note: no iGPU - GPU passthrough is all-or-nothing (host must be headless)

### IOMMU groups (GPU subtree, PLX switch)

| Group | Bus | Device |
|-------|-----|--------|
| 82 | 02:00.0 | Navi 10 XL Upstream Port (PCIe switch) |
| 83 | 03:00.0 | Navi 10 XL Downstream Port |
| 84 | 04:00.0 | Radeon RX 5700 XT (VGA) |
| 85 | 04:00.1 | Navi 10 HDMI Audio |

Pass-through set for the desktop VM: 03:00.0 + 04:00.0 + 04:00.1.

## Storage

| Device | Model | Capacity | Connection | Role (plan) |
|--------|-------|----------|-----------|-------------|
| /dev/nvme0n1 | Lexar SSD NM710 500GB | 465.8G | NVMe (PCIe 3.0 on this board) | PVE root + ISO |
| /dev/sda | WDC WDS100T2B0B-00YS70 (WD Blue 3D) | 931.5G | SATA | VM disk pool (LVM-thin) |
| /dev/sdb | ST1000LM035-1RK172 (Seagate BarraCuda, 5400 rpm) | 931.5G | SATA | ZFS mirror (NAS) + sdc |
| /dev/sdc | ST1000LM049-2GH172 (Seagate BarraCuda, 7200 rpm) | 931.5G | SATA | ZFS mirror (NAS) + sdb |
| /dev/sdd | PNY CS900 1TB SATA SSD | 931.5G | **USB SATA** | PBS / detached backup ("Hdscythe") |

- SATA controllers: C610/X99 6-port AHCI (00:1f.2) + sSATA (00:11.4)
- NVMe controller: Longsys (01:00.0) - Lexar NM790/Viper VP4300 Lite class

## Network

- **eth**: Realtek RTL8111/8168/8411 1GbE (07:00.0, r8169) - primary uplink
- **wlan**: Broadcom BCM4360 802.11ac (06:00.0) - backup uplink, not a passthrough target
- Planned MGMT: **10.10.30.1** on 10.10.30.0/24 - `https://10.10.30.1:8006`
- Internal VM bridge: vmbr0 10.30.0.0/24 - gw 10.30.0.1

## USB / Misc

- Standard X99 ports (xHCI/EHCI); no USB4/Thunderbolt
- On-board HD audio + HDMI audio via GPU
- No IPMI/BMC - manual power cycling only; hardware watchdog iTCO_wdt available

## VMs / Containers (planned)

- **QEMU VMs**: desktop VM (RX 5700 XT passthrough), misc services
- **LXC containers**: AdGuard/Pi-hole, media stack, PBS (or LXC/VM), etc.
- None exist yet - host is still a workstation (see `proxlab-notes.md`)

## References

- Plan/spec: [../plan/proxlab.md](../plan/proxlab.md)
- Platform + passthrough research: [../research/24-dual-socket-x99-pve.md](../research/24-dual-socket-x99-pve.md), [../research/25-amd-rx5700xt-passthrough.md](../research/25-amd-rx5700xt-passthrough.md)
- Storage design: [../research/26-proxlab-storage-nas.md](../research/26-proxlab-storage-nas.md)