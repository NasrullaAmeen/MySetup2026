---
tags: [research, omalaptop, arrowlake, igpu, sriov, gvt-g, xe, intel, fallback]
created: 2026-09-18 17:00:45 +05
modified: 2026-09-18 18:25:03 +05
---

# Arrow Lake iGPU (8086:7d67) realities - why it stays host-only

The OmaLaptop pivot calls for the Arrow Lake iGPU to drive only the Omarchy
desktop, never a VM. This note collects current (2026) research that
validates that hard rule and kills the old "share iGPU with a VM" idea.

## TL;DR

- GVT-g is dead (archived upstream).
- SR-IOV VF passthrough of this iGPU is BROKEN on Arrow Lake today
  (VFIO_MAP_DMA -EINVAL), on both i915-sriov-dkms and the xe path.
- Two concurrent GPU passthroughs on Z890/Arrow Lake can hard-lock the host
  (PVE forum report). We run exactly one dGPU passthrough - do not add the
  iGPU to any VM.
- Conclusion: iGPU = host desktop only. MainWin11 gets virtio-gpu, not the
  iGPU (plan already says this).

## GVT-g (Intel GVT-g mediated passthrough)

- Archived upstream (security escape classes; only Gen 5-10 supported).
- Do not plan around it for Arrow Lake - there is no GVT-g for 12th gen+.

## SR-IOV on Arrow Lake (Series 2 ARL)

- Intel documented SR-IOV support on Arrow Lake with a Linux host and
  Windows/Linux guest, BUT:
  - Mainline `i915` does not enable SR-IOV on ARL; you need the community
    `i915-sriov-dkms` module;
  - kernel params for that path:
    `intel_iommu=on i915.enable_guc=3 i915.max_vfs=7 i915.force_probe=7d67
    module_blacklist=xe`;
  - the `xe` driver does NOT support SR-IOV on MTL/ARL (G2H error);
    upstream "not yet ready to support SR-IOV on MTL/ARL".

## The blocker for this laptop (7d67)

- Community reports reproduce `VFIO_MAP_DMA failed: Invalid argument`
  (-EINVAL) when mapping the VF's own MMIO BAR at a high 64-bit address
  (e.g. 0x380000000000) on both 8086:7d67 (our exact device) and 7dd1
  (strongtz/i915-sriov-dkms issue #359).
- So even with VFs created, handing one to a VM fails. This is an upstream
  DRM/xe bug, not a config issue.

## Concurrent-passthrough lockup risk (Z890/ARL)

- PVE forum: a build passing TWO GPUs (dGPU + iGPU) into VMs at once
  hard-locked the host on ASUS Z890 + Arrow Lake (PVE 9.2.3).
- For us: exactly one dGPU is ever in a VM (GameWin11, sometimes swapped to
  MainWin11). Keep iGPU permanently on the host - avoids the class of bug.

## efifb / sysfb console binding

- 00:02.0 keeps the kernel console/efifb/SysFB attachment; that is fine for
  a desktop host but means the iGPU must stay the primary display. Never
  hide/blacklist i915 on this machine.

## Driver forks

- `i915.force_probe=7d67` was required on older kernels; recent mainline
  includes ARL IDs. On Omarchy (rolling Arch kernel) i915 should bind
  natively; `module_blacklist=xe` only needed if the xe probe path is taken.

## Sources

- strongtz/i915-sriov-dkms issue #359 (VFIO_MAP_DMA -EINVAL on 7d67/7dd1).
- PVE forum ARL/Z890 two-GPU passthrough hard-lock report (PVE 9.2.3).
- Intel MTL/ARL SR-IOV upstream status (G2H / "not yet ready").
- research/02-* and plan/omalaptop.md hard rules already encode "host only".

## Applied to / next steps

- Phase A: after Omarchy install, confirm i915 owns 00:02.0 and dmesg shows
  no xe gag; keep `module_blacklist=xe` if needed.
- Never attach `hostdev 0000:00:02.0` to any VM XML.
- Phase C9-10: MainWin11 stays virtio-gpu (documented swap script only for
  the dGPU, research/29).