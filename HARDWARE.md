# HARDWARE.md — Machine spec (as of 2026-09-17)

## System
| | |
|---|---|
| Vendor / Model | Acer Predator **PHN16S-71** (laptop) |
| Board | ARL FTYPE_ARX (Intel HM870 chipset) |
| BIOS | INSYDE V1.21 (UEFI, Secure Boot **enabled**, TPM **2.0**) |
| OS now | Omarchy (Arch), kernel 7.2.3-arch1-3, PREEMPT_DYNAMIC, x86_64 |

## CPU
- Intel **Core Ultra 9 275HX** (Arrow Lake-HX), 24C/24T (8P + 16E, no HT), 1 socket
- 800 MHz – 5400 MHz, L2 40 MiB, L3 36 MiB, single NUMA node
- Flags: VT-x, **VT-d** (ept, vpid, flexpriority), AVX2, AES-NI, SHA-NI, AVX-VNNI

## Memory
- 64 GB DDR5 total (62 GiB usable, 2x SODIMM — two SPD EEPROMs present)
- Swap: zram 62.2 G + kernel `resume=` (hibernation image in LUKS/btrfs)

## Storage
| Device | Size | Model | Connection | Content |
|---|---|---|---|---|
| nvme1n1 | 953.9G | SK hynix HFS001TEJ9X125N (P41 OEM) | internal M.2 slot 1 | p1: EFI 2G /boot · p2: LUKS → btrfs (@, @home, @log) — **Omarchy, booted** |
| nvme0n1 | 238.5G | WDC PC SN520 SDAPNUW-256G-1002 | internal M.2 slot 2 | p1: EFI · p2: MSR · p3: NTFS "Basic data partition" — **Windows** |
| sdb | 931.5G | PNY CS900 1TB SATA SSD | **USB SATA** | btrfs "Hdscythe" (mounted /run/media/darko/Hdscythe) |

**Two internal M.2 slots** (both 2280, PCIe Gen4 x4), now BOTH populated:
1. SK hynix P41 (1TB) — live Omarchy.
2. WD SN520 (256GB) — Windows NTFS install. Budget-tier x2-lane OEM drive.

The M.2 claim above (Acer officially supports up to 2TB/slot) still stands if slot 2
gets upgraded.

## GPUs (2)
| Bus | Device | Class | IOMMU grp | Driver |
|---|---|---|---|---|
| 00:02.0 | Intel Arrow Lake-S UHD (8086:7d67) | iGPU | **0** (alone) | i915 (xe alt) |
| 02:00.0 | NVIDIA **GeForce RTX 5060 Max-Q / Mobile** GB206M (10de:2d59) | dGPU | **14** (GPU+audio, clean) | nvidia / nvidia_drm |

- dGPU HDA audio in same IOMMU group (14) — passes as a block.
- Panel likely iGPU-routed (Optimus); dGPU ports for external display.

## Networking
- Ethernet: **Realtek Killer E3000 2.5GbE** (82:00.0, driver r8169)
- Wi-Fi: Intel CNVi 800-series (80:14.3, iwlwifi) 2.4/5/6GHz, ax, 2x2 — currently 5GHz 80MHz
- Bluetooth: Intel (hci0)
- USB LAN: Realtek RTL8153 1GbE (0bda:8153, via hub)

## Audio
- SOF HDA DSP (Intel ACE, sof-hda-dsp) — main
- NVIDIA HDA (01:00.1) — HDMI/DP via dGPU

## USB / Thunderbolt
- **2x USB4/TB4** ports (ucsi USBC000:001/002, TB controller "Gen14")
- xHCI roots + internal hubs: VIA 2109/2822, Realtek 0bda:0411/5411, VL 2109:8884 Billboard
- Peripherals: USB keyboard (1c4f:0084), 2.4G dongle (25a7:fa61), cameras:
  - SunplusIT USB 2.0 Camera (0806:0806)
  - ACER FHD User Facing (04f2:b831)

## IOMMU (VT-d active, 25 groups — after second M.2 populated)
- 00:02.0 iGPU → group 0, alone ✅
- 00:01.0 root port + **WD SN520** (01:00.0) → group 13, alone ✅
- 00:06.0 root port + **RTX 5060 + HDA** (02:00.0/02:00.1) → group 14, alone ✅
- 00:06.4 root port + **hynix P41** (03:00.0) → group 15, alone ✅
- Remaining chipset devices (wifi, audio, TB, xHCI) on separate groups.

Every passthrough target (iGPU, dGPU, both NVMe) sits in its own IOMMU group.

## Sensors / thermal (idle-ish sample)
- Package ~79 °C (crit 105), SPD DIMM 50–59 °C, NVMe 49–55 °C, LAN 49.5 °C
- Battery: SIMPLO AP24A7Q, 100%, 29 cycles (cooled, effectively desktop-use)

---

## Impact on the ProxDev plan (laptop constraints)

1. **Both internal M.2 slots are now busy** — slot 1 = Omarchy (hynix 1TB),
   slot 2 = Windows (WD SN520 256GB). PVE currently has **no dedicated home**.
   Options:
   - Repurpose slot 2: wipe the WD's Windows, install PVE there. 256GB is
     enough for PVE OS + ISOs, but VM disks then need a shared/secondary home
     (hynix carve or USB4 storage).
   - Upgrade slot 2 to a 1–2TB Gen4 drive and install PVE there (cleanest
     long-term); use the WD elsewhere.
   - Keep WD as a Windows disk → PVE must live external (TB4/USB4) or share
     the hynix.
2. **dGPU is laptop-grade + iGPU-routed panel.** RTX 5060 passthrough to GameWin11
   works (clean group 14), but internal-panel output will stay on the iGPU; use
   dGPU ports for gaming display.
3. **Secure Boot is on.** PVE (GRUB/systemd-boot) needs SB keys enrolled or SB off.
4. **Thermals/battery**: always-on MainOS + 6 VMs on a laptop = heat + fan + power.
   Realistic target: 2–3 VMs on battery, rest docked/AC.
5. **IOMMU is clean** — iGPU grp0, dGPU grp14, and both NVMe are individually
   isolated: the best-case start for VFIO.
6. **Backup target**: PNY 1TB SATA (btrfs "Hdscythe") is the **dedicated** Proxmox
   Backup Store / VM snapshot target (detached between runs).