---
tags: [proxmox, hardware, server, inventory]
created: 2026-09-17 20:39:00 +05
modified: 2026-09-17 21:18:51 +05
---

# ProxDev - Full Hardware Stack

## Summary

| Component | Model / Detail |
|-----------|----------------|
| CPU | Intel Core Ultra 9 275HX - 24 cores / 24 threads, 1 socket |
| RAM | 62.2 GB total (2.4 GB used), 8 GB swap |
| GPU (dGPU) | NVIDIA GeForce RTX 5060 Max-Q / Mobile (GB206M) |
| GPU (iGPU) | Intel Arrow Lake integrated Xe |
| NVMe 1 | WDC PC SN520 256 GB (wearout 96%) |
| NVMe 2 | SK hynix HFS001TEJ9X125N 1 TB (wearout 99%) |
| NIC | Realtek RTL8125 2.5GbE + 3x onboard eth + Intel WiFi |
| Storage pools | local (67.7 GB) + local-lvm (141.2 GB) |

## Platform

- **Node**: ProxDev (10.10.10.10)
- **Proxmox VE**: pve-manager 9.2.20 (release 9.2)
- **Kernel**: Linux 7.0.2-6-pve, x86_64
- **Boot**: EFI mode, Secure Boot disabled
- **Uptime (at scan)**: ~52 min

## CPU

- **Model**: Intel Core Ultra 9 275HX
- **Sockets**: 1 , **Cores**: 24 , **Threads**: 24
- **Family**: 6 (Arrow Lake-HX)
- Notes: vmx/hvm (VT-x), ept, aes, avx2/avx_vnni, gfni, vaes, sha_ni, pt, smap/smep, umip, rdt_a

## Memory

- **Total RAM**: 62.2 GB (66,819,158,016 B)
- **Used**: 2.4 GB , **Free**: 56.1 GB
- **Swap**: 8 GB (0 used)

## GPU / Display

- **iGPU**: Intel 0x8086:0x7d67 (Arrow Lake integrated Xe)
- **dGPU**: NVIDIA 0x10de:0x2d59 - GB206M [GeForce RTX 5060 Max-Q / Mobile]
- **HDMI/DP audio**: NVIDIA 0x10de:0x22eb (part of RTX 5060)

## Storage (NVMe)

| Device | Model | Capacity | Health | Wearout |
|--------|-------|----------|--------|---------|
| /dev/nvme0n1 | WDC PC SN520 SDAPNUW-256G-1002 | 256 GB | PASSED | 96% |
| /dev/nvme1n1 | SK hynix HFS001TEJ9X125N | 1.0 TB | PASSED | 99% |

### Storage pools

- **local** (dir): 72.7 GB - rootfs (backup, iso, vztmpl) - 8.6% used
- **local-lvm** (lvmthin): 151.6 GB - VM disk images - 0% used

## Network

- **nic1**: eth - active (main uplink)
- **nic0**: eth - USB adapter (altname enx74d4ddcda898)
- **nic2 / nic3**: onboard ethernet
- **wlp128s20f3**: WiFi (Intel 0x8086:0x7f70, alt wlxec8e771cc45b)
- Dedicated NIC: Realtek RTL8125 2.5GbE (0x10ec:0x3000 at 82:00.0)

## Expansion - PCI devices (full bus map)

| Bus | Class | Vendor:Device | Description |
|-----|-------|---------------|-------------|
| 00:02.0 | VGA | 8086:7d67 | Intel Arrow Lake iGPU |
| 00:04.0 | Signal proc | 8086:ad03 | Intel IPU (imaging) |
| 00:08.0 | System | 8086:ae4c | Intel telemetry/PM |
| 00:0a.0 | Signal proc | 8086:ad0d | Intel (media/ISP) |
| 00:0b.0 | Signal proc | 8086:ad1d | Intel DMA/accel |
| 00:0d.0 | USB 3 | 8086:7ec0 | Intel xHCI |
| 00:0d.2 | USB 3 | 8086:7ec2 | Intel xHCI host |
| 00:1f.5 | Serial bus | 8086:ae23 | Intel SPI/serial I/O |
| 01:00.0 | NVMe | 15b7:5003 | SanDisk/WD - 256 GB SN520 |
| 02:00.0 | VGA | 10de:2d59 | NVIDIA RTX 5060 Max-Q (GB206M) |
| 02:00.1 | Audio | 10de:22eb | NVIDIA HDMI/DP audio |
| 03:00.0 | NVMe | 1c5c:1959 | SK hynix - 1 TB SSD |
| 80:14.0 | USB 3 | 8086:7f6e | Intel xHCI (PCH) |
| 80:14.3 | Network | 8086:7f70 | Intel WiFi (BE200-class) |
| 80:14.5 | Security | 8086:7f2f | Intel cryptographic/SE |
| 80:15.0/.1/.3 | Serial bus | 8086:7f4c/7f4d/7f4f | Intel SIO/CNVi |
| 80:16.0 | Comm | 8086:7f68 | Intel HECI (MEI) |
| 80:19.0/.1/.2 | PCIe | 8086:7f7a/7f7b/7f5c | Intel PCIe root ports |
| 80:1f.3 | Audio | 8086:7f50 | Intel HD Audio (PCH) |
| 80:1f.4 | SMBus | 8086:7f23 | Intel SMBus |
| 80:1f.5 | Serial | 8086:7f24 | Intel SPI controller |
| 81:00.0 | SD host | 10ec:522a | Realtek card reader |
| 82:00.0 | Ethernet | 10ec:3000 | Realtek RTL8125 2.5GbE |

## USB

- 4 controller buses (16 devices total); no controllers passed through to guests

## VMs / Containers

- **QEMU VMs**: none
- **LXC containers**: none