---
tags: [research, power, efficiency, arrow-lake, 275hx, aspm, cstates, idle]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Power & Efficiency on Core Ultra 9 275HX

Research date: 2026-09-17. HW context: 275HX (8P+16E, 24c/24t), 62GB RAM, 2x NVMe, RTL8125, RTX 5060 Max-Q (compute only).

## Arrow Lake idle story

- Arrow Lake (3nm compute tile + NPU) has a genuinely higher idle floor than AMD; expect a 25W+ platform at best.
- Deep package C10 needs ALL IA cores in C10 + iGPU in RC6 + VRs at PS4 + crystal clock off + every device honoring LTR. A 10G/25G NIC, hard disks, or the dGPU awake blocks C6/C10.
- Newer microcode/BIOS revisions reportedly raise idle draw (consider rolling back if nothing changed).

## Kernel

- Native intel_idle Arrow Lake tables landed in kernel 6.13; older kernels (PVE 6.8/6.12) fall back to ACPI _CST and cap at C3 - the #1 reason for ~55W idle.
- Use a PVE kernel with native Arrow Lake pmc/DMU fixes (6.14+/6.17+ HWE paths).
- Verify C10: `cat /sys/devices/system/cpu/cpu*/cpuidle/state*/name`. Keep intel_idle default.
- intel_pstate active + HWP: the real knob is EPP, not governor. `cpupower frequency-set -g powersave`, add `pcie_aspm=force` on cmdline.

## NVIDIA RTX 5060 Max-Q (compute-only)

- D0/P8 idle ~4W (nvidia-smi); real sleep needs RTD3 (NVreg_DynamicPowerManagement=0x02/0x03) + udev power/control=auto -> D3cold. On desktop boards RTD3 is disabled by default.
- Blackwell 5060: nvidia_drm (use fbdev=0) and nvidia-persistenced block D3cold; nvidia-smi polling loops wake it - poll power/runtime_status instead.
- Cleanest for compute-only: do NOT bind nvidia on the host at all (VFIO/passthrough) so the card sits in D3. [CAUTION] Pass it to a VM if you want host-side vGPU/NVENC - NvFBC caveats apply.

## ASPM

- `pcie_aspm=force`; check powertop Device Stats for 100%-active devices.
- RTL8125: r8169 disables L1 ASPM by default; use the r8125 DKMS driver (ASPM on by default) or force /sys/.../link/l1_aspm=1 via boot service. Keep L0s off (quirk; link corruption).

## Practical

- `powertop --auto-tune` gained Arrow Lake support in v2.15-4 (2025); saves ~3W here.
- `tuned-adm profile server-powersave` - avoid plain `powersave` on Proxmox (reported NIC/instability).
- Undervolting: Plundervolt mitigations lock most Arrow Lake BIOSes; intel-undervolt does not cover Arrow Lake. Skip - use EPP instead.
- Disabling boost on a compute box is a bad trade; use EPP.

## Realistic numbers

| Scenario | Idle watts |
|----------|-----------|
| 285HX + 25GbE (measured, Windows) | 27.8W (17.8W w/o NIC) |
| 275HX-class PVE stock | ~55W |
| 275HX-class tuned (C10 + new kernel) | ~42W |
| + powertop/powersave | -3-4W |
| 285H mini PCs (best case) | 12-13W |

Budget: 2x NVMe ~2W, RTL8125 ~1-2W, dGPU 0W (D3) to 4W (D0). Expect 30-45W out of the box, 22-32W tuned on a desktop-HX board; 8-20W only if PC10 + dGPU asleep + iGPU-only + Intel NICs set.

## Checklist

- BIOS: Package C-state limit C10, Enhanced C-states ON, ASPM L1.1/L1.2 on all slots/bridges, disable VMD, iGPU on (RC6 for PC10), dGPU slot ASPM L1, no "unbounded" PL if noise matters.
- Kernel: PVE 6.17+/6.19 (native intel_idle + Arrow Lake pmc), pcie_aspm=force, keep intel_idle, verify C10.
- Driver: r8125 DKMS or l1_aspm sysfs fix; NVIDIA NVreg_DynamicPowerManagement=0x02, nvidia_drm.fbdev=0, udev power/control=auto, mask nvidia-persistenced/nvidia-powerd, don't poll nvidia-smi.
- Userspace: tuned server-powersave, cpupower -g powersave, EPP balance_performance, powertop --auto-tune.
- Measure at the wall (plug meter).

## Sources

- https://forum.proxmox.com/threads/intel_idle-c-states-in-proxmox-with-intel-series-200-cpus.179143/
- https://www.kernel.org/doc/html/v6.19/admin-guide/pm/intel_pstate.html
- https://github.com/NVIDIA/open-gpu-kernel-modules/issues/980
- https://00l.uk/2026/04/05/enabling-aspm-on-a-realtek-rtl8125b-2-5gbe-nic-in-proxmox-linux/
- https://www.linuxlinks.com/minisforum-ms-02-ultra-285hx-running-linux-power-consumption/
- https://zlendy.com/blog/psa-tuned-in-proxmox-ve