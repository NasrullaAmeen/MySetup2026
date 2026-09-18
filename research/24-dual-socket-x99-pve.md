---
tags: [research, proxlab, x99, dual-socket, xeon, broadwell-ep, iommu, platform]
created: 2026-09-18 10:15:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 24. X99 dual-socket (Broadwell-EP) as a Proxmox host

Research date: 2026-09-18. Target: **ProxLab** = the X99 workstation
(`omarchy`, 2x Xeon E5-2660 v4, C610/X99 chipset). Platform notes for
running Proxmox VE on it. Hardware captured: [../plan/proxlab.md](../plan/proxlab.md).

## Why this platform is interesting for a homelab

- Cheapest genuine dual-socket ECC compute on the second-hand market.
- 28C/56T of older-but-solid Broadwell cores; Tons of RAM capacity (up to
  1 TB, ECC RDIMM), and 40 PCIe 3.0 lanes per socket worth of slots.
- Old hardware: weak AVX-512 (none), PCIe 3.0, DDR4-2400 max, ~105 W TDP per
  socket. Fine for PVE/VMs/LXC, not for AI inference or fast NVMe.

## Observed hardware (read off this box, 2026-09-18)

| Component | Value |
|---|---|
| Board | X99 (dual LGA2011-3, C610/X99 chipset) |
| CPU | 2x Intel Xeon E5-2660 v4 (Broadwell-EP), 14C/28T, 2.0/3.2 GHz, 35 MiB L3, TDP 105 W |
| RAM | 109 GiB DDR4 ECC RDIMM (4ch per socket, 8 channels total), zram 109.9 G swap |
| Storage | Lexar NM710 500G NVMe + WD Blue 1T SATA SSD + 2x Seagate 1T HDD + PNY 1T USB |
| GPU | Gigabyte RX 5700 XT Gaming OC (Navi 10, 04:00.0) + HDMI audio (04:00.1) |
| Net | Realtek RTL8168 1G (07:00.0) + Broadcom BCM4360 ac (06:00.0) |
| IOMMU | VT-d active: 92 groups |

## NUMA: one node exposed (verify before sizing VMs)

- `lscpu` reports a **single NUMA node** (0-55) despite two sockets. Reason:
  NUMA *cluster* mode / SNC is typically off on these boards and the firmware
  exposes memory as one interleaved domain. Whatever the cause:
  - Run `numactl --hardware` to confirm real topology.
  - If two sockets show as one NUMA node, inter-socket QPI traffic is hidden
    from the scheduler - fine for light VM loads, suboptimal for large
    cross-socket VMs.
  - For perf-sensitive VMs give each VM 8-14 cores and rely on `cpuset`
    affinity; keep VM count modest so both sockets get used.
- BIOS: leave NUMA (SNC) off unless you want two nodes; PVE handles either.

## Memory and ECC

- RDIMM ECC is a real upgrade over the laptop (OmaLaptop). Verify it actually
  reports errors:
  - `dmesg | grep -i edac`
  - `rasdaemon` (or `edac-util`) to log CE/UE counts; install
    `rasdaemon` and enable `ras-mc-ctl`.
- Note the observed "109 GiB usable" from `free -h` - this includes reserved
  ECC/overhead; plan VM pools against ~100 GiB to be safe.
- 8 memory channels (4/socket): populate all channels if you can; interleave
  is automatic.

## IOMMU and passthrough

- 92 groups observed, key isolation (from live `lspci`):

| IOMMU group | Device |
|---|---|
| 82 | 02:00.0 Navi 10 XL Upstream Port (PCIe switch) |
| 83 | 03:00.0 Navi 10 XL Downstream Port |
| 84 | 04:00.0 RX 5700 XT (VGA) |
| 85 | 04:00.1 Navi 10 HDMI Audio |
| (own) | 01:00.0 Lexar NVMe, 07:00.0 RTL8168, 06:00.0 BCM4360 |

- The GPU sits behind a **PLX PCIe switch**: the upstream port (82), switch,
  and the GPU + audio (84/85) land in separate groups. To passthrough cleanly
  you attach the whole subtree: pass 03:00.0 (bridge) + 04:00.0 + 04:00.1, and
  `pveperf`/`lspci` will confirm. Do **not** pass 02:00.0 (upstream) - it is
  the switch's connection to the root complex.
- `intel_iommu=on`/`amd_iommu=on` are kernel **defaults since 6.8**, and VT-d
  is clearly active here. A kernel arg is only needed if you want
  `iommu=pt` (recommended to reduce IOMMU translation overhead).
- If a PCI device ever ends up sharing a group with a sibling you want to
  keep on the host, only then consider ACS override patches (not needed for
  this box).

## Power / thermals

- Dual Xeon idle draw is the pain point: expect **90-140 W** at the wall with
  two E5-2660 v4, C610 chipset, 3 spinning/e-SATA disks. Add 3x 1T HDDs and
  it climbs. Measured later once PVE is on - track via a smart plug or
  `powerstat`on the running host.
- `intel_pstate` best with `performance` or `powersave` governors; C-states:
  Broadwell-EP supports package C-states poorly deep - expect CPUs to sit in
  C1e mostly. Do not fight for idle watts here; this is an always-on homelab
  box.
- HDDs: spin down via `hdparm`/`sdparm` only for the two Seagate bulk disks;
  keep the WD SSD awake (small draw).

## Watchdog / BMC

- C610/X99 has **iTCO_wdt** (Intel TCO). For Proxmox HA/fencing later this is
  a real hardware watchdog (better than `softdog`). Select `iTCO_wdt` in
  `/etc/default/pve-ha-manager` only if clustering (see 27-*).
- No IPMI/BMC on these boards - no out-of-band power control; plan manual
  power cycling for maintenance.

## Verdict for ProxLab

- Perfectly capable PVE host: lots of RAM/cores, clean IOMMU, hardware
  watchdog. Weak spots: PCIe 3.0, no iGPU (GPU passthrough is all-or-nothing),
  idle power. Config: PVE on NVMe (small, fast), VM pool on SATA SSD, bulk
  data on the two HDDs (mirror). See [26-*](./26-proxlab-storage-nas.md).