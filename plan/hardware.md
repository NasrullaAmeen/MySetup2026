---
tags: [plan, hardware, laptop, proxmox, iommu]
created: 2026-09-17 15:59:00 +05
modified: 2026-09-18 09:40:00 +05
---

# Hardware - machine spec (as of 2026-09-17)

## System

| | |
|---|---|
| Vendor / Model | Acer Predator **PHN16S-71** (laptop) |
| Board | ARL FTYPE_ARX (Intel HM870 chipset) |
| BIOS | INSYDE V1.21 (UEFI, Secure Boot **enabled**, TPM **2.0**) |
| OS now | Proxmox VE 9.2.20 on nvme0n1 (WD); Omarchy (Arch) moved into MainArch VM |

## CPU

- Intel **Core Ultra 9 275HX** (Arrow Lake-HX), 24C/24T (8P + 16E, no HT), 1 socket
- 800 - 5400 MHz, L2 40 MiB, L3 36 MiB, single NUMA node
- Flags: VT-x, **VT-d** (ept, vpid, flexpriority), AVX2, AES-NI, SHA-NI, AVX-VNNI

## Memory

- 64 GB DDR5 total (62 GiB usable, 2x SODIMM - two SPD EEPROMs present)
- Swap: zram 62.2 G + kernel `resume=` (hibernation image in LUKS/btrfs)

## Storage

| Device | Size | Model | Connection | Content |
|---|---|---|---|---|
| nvme1n1 | 953.9G | SK hynix HFS001TEJ9X125N (P41 OEM) | internal M.2 slot 1 (Gen4 x4) | **VM disk pool** (LVM-thin) |
| nvme0n1 | 238.5G | WDC PC SN520 SDAPNUW-256G-1002 | internal M.2 slot 2 (Gen4 x4) | **Proxmox VE** root + EFI + ISO (Windows partition repurposed) |
| sdb | 931.5G | PNY CS900 1TB SATA SSD | **USB SATA** | **Backup target** (PBS / rsync) - detached between runs |

**Two internal M.2 slots** (both 2280, PCIe Gen4 x4), both populated:

1. SK hynix P41 (1TB) - VM disks (was live Omarchy).
2. WD SN520 (256GB) - Proxmox VE (was Windows NTFS). Budget-tier x2-lane OEM drive,
   fine for PVE OS + ISO.

The M.2 claim above (Acer officially supports up to 2TB/slot) still stands if slot 2
gets upgraded.

## GPUs (2)

| Bus | Device | Class | IOMMU grp | Driver |
|---|---|---|---|---|
| 00:02.0 | Intel Arrow Lake-S UHD (8086:7d67) | iGPU | **0** (alone) | i915 (xe alt) |
| 02:00.0 | NVIDIA **GeForce RTX 5060 Max-Q / Mobile** GB206M (10de:2d59) | dGPU | **14** (GPU+audio, clean) | nvidia / nvidia_drm |

- dGPU HDA audio in same IOMMU group (14) - passes as a block.
- Panel likely iGPU-routed (Optimus); dGPU ports for external display.

## Networking

- Ethernet: **Realtek Killer E3000 2.5GbE** (82:00.0, driver r8169)
- Wi-Fi: Intel CNVi 800-series (80:14.3, iwlwifi) 2.4/5/6GHz, ax, 2x2 - currently 5GHz 80MHz
- Bluetooth: Intel (hci0)
- USB LAN: Realtek RTL8153 1GbE (0bda:8153, via hub)

## Audio

- SOF HDA DSP (Intel ACE, sof-hda-dsp) - main
- NVIDIA HDA (01:00.1) - HDMI/DP via dGPU

## USB / Thunderbolt

- **2x USB4/TB4** ports (ucsi USBC000:001/002, TB controller "Gen14")
- xHCI roots + internal hubs: VIA 2109/2822, Realtek 0bda:0411/5411, VL 2109:8884 Billboard
- Peripherals: USB keyboard (1c4f:0084), 2.4G dongle (25a7:fa61), cameras:
  - SunplusIT USB 2.0 Camera (0806:0806)
  - ACER FHD User Facing (04f2:b831)

## IOMMU (VT-d active, 25 groups - after second M.2 populated)

- 00:02.0 iGPU -> group 0, alone - passthrough clean.
- 00:01.0 root port + **WD SN520** (01:00.0) -> group 13, alone.
- 00:06.0 root port + **RTX 5060 + HDA** (02:00.0/02:00.1) -> group 14, alone.
- 00:06.4 root port + **hynix P41** (03:00.0) -> group 15, alone.
- Remaining chipset devices (wifi, audio, TB, xHCI) on separate groups.

Every passthrough target (iGPU, dGPU, both NVMe) sits in its own IOMMU group.

## Sensors / thermal (idle-ish sample)

- Package ~79 C (crit 105), SPD DIMM 50-59 C, NVMe 49-55 C, LAN 49.5 C
- Battery: SIMPLO AP24A7Q, 100%, 29 cycles (running docked/AC as a server)

---

## Impact on the ProxDev plan (laptop constraints)

1. **Both internal M.2 slots are busy and allocated** - slot 1 = VM pool
   (hynix P41 1TB), slot 2 = Proxmox VE (WD SN520 256GB). 256GB is enough for
   PVE OS + ISOs; all VM disks live on the hynix pool. Upgrade path: swap slot 2
   for a 1-2TB Gen4 drive and carve VM space there; use the WD as USB storage.
2. **dGPU is laptop-grade + iGPU-routed panel.** RTX 5060 passthrough to GameWin11
   works (clean group 14), but internal-panel output will stay on the iGPU; use
   dGPU ports for gaming display.
3. **Secure Boot was on.** PVE would need SB keys enrolled or SB off. (Status on
   the running box: verify how PVE 9.2.20 booted - systemd-boot SB.)
4. **Thermals/battery**: always-on MainOS + VMs on a laptop = heat + fan + power.
   Realistic target: 2-3 VMs on battery, rest docked/AC.
5. **IOMMU is clean** - iGPU grp0, dGPU grp14, and both NVMe are individually
   isolated: the best-case start for VFIO.
6. **Backup target**: PNY 1TB SATA (btrfs "Hdscythe") is the **dedicated** Proxmox
   Backup Store / VM snapshot target (detached between runs).