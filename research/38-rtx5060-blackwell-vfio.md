---
tags: [research, omalaptop, blackwell, vfio, gpu, passthrough, rtx5060, d3cold, iommu]
created: 2026-09-18 17:00:45 +05
modified: 2026-09-18 18:25:03 +05
---

# RTX 5060 (Blackwell) VFIO passthrough - current pitfalls and fixes

Target of the outer GameWin11 VM: 10de:2d59 (RTX 5060 / GB206, Max-Q) +
10de:22eb (HDMI/audio), IOMMU group 14. This note upgrades research/02 with
2026 findings. Big new lead: DO NOT use `iommu=pt` on Blackwell.

## Hard finding 1 - Blackwell reset is BROKEN

- "RTX 50 (Blackwell) reset is broken; one VM run per host boot. NVIDIA has
  acknowledged the issue and has no clean fix" (ArchWorks 2026 guide).
- Matches research/02: GSP firmware keeps Write-Protected Region 2 (WPR2)
  across PCIe FLR -> GPU dead on the 2nd VM start of a session.
- Consequence for us on the laptop: boot -> start GameWin11 once -> run games
  -> reboot when you need the GPU in another VM. Encode this as a runbook
  rule and a suspend gate, not a fix.
- vendor-reset DKMS is AMD-only; it does NOT help NVIDIA. Do not use it.

## Hard finding 2 - Blackwell rejects `iommu=pt`

- Blackwell firmware requests a 1:1 (identity) IOMMU mapping. With the
  kernel running `iommu=pt`, vfio_pci rejects the device at VM start:
  `"Firmware has requested this device have a 1:1 IOMMU mapping, rejecting..."`
  and QEMU exits immediately.
- So: do NOT set `iommu=pt` on the Omarchy host. Just `intel_iommu=on`
  (which is already the kernel default since 6.8). We had `iommu=pt` in
  plan/omalaptop.md and docs/tasks.md - removing now (see files modified).
- (If a future problem appears, a patched kernel/workaround exists that
  comments out `return -EINVAL` in vfio_pci_core.c, but the clean fix on
  Arch/Omarchy is simply to drop `iommu=pt`.)

## Hard finding 3 - idle D3 cold hangs (5060 Ti reports)

- Most-reported 5060-class freeze: the adapter drops to D3cold while in
  VFIO and the host hangs or the VM start wedges.
- Mitigations (use both):
  - kernel cmdline: `vfio-pci.disable_idle_d3=1` (keep device out of D3);
  - udev rule keeping it in D0 (d3cold_allowed = 0) and power/control auto
    off. Cost: higher idle power draw (the GPU sleeps less).
- On a laptop this also avoids suspend/resume landmines when the dGPU is
  idle in VFIO. Plan Phase C step 9 already lists "reset quirk if needed".

## Firmware / GOP update

- NVIDIA provides a vBIOS/GOP firmware updater; a stale GOP can cause the
  reset-and-hang loop on Blackwell. Run the update tool inside the passed
  VM (or from Windows) before declaring passthrough broken.

## Config deltas vs older notes (research/02) that still hold

- Blacklist nouveau + nvidia + nvidiafb + nvidia-gpu in the host
  (`/etc/modprobe.d/blacklist.conf`), softdep `snd_hda_intel pre: vfio-pci`,
  and bind `vfio-pci.ids=10de:2d59,10de:22eb` early (host never binds the
  dGPU since it is 100% VFIO - dual-GPU, clean split).
- No need for vendor-id hiding / `kvm hidden` masking; modern NVIDIA drivers
  run fine in VMs. `-cpu host` + hyperv enlightenments as in research/35.
- `pcie_aspm=off` on laptop if the dGPU stalls after reboot; keep `Above 4G
  Decoding / ReBAR` on in BIOS.
- PVE-kernel regressions (research/02 line 53) are PVE-specific; the Omarchy
  Arch kernel sidesteps PVE's 6.17/7.0 bugs, but check the forum + Arch
  kernel notes before a big `pacman -Syu` if passthrough misbehaves.

## One-VM-per-boot workflow (to encode in plan Phase C)

1. Boot host -> dGPU already in VFIO.
2. Start GameWin11 -> games -> shutdown GameWin11 gracefully.
3. GPU may not reset -> if need GPU again: host reboot, not a workaround.
4. MainWin11 swap: same reboot rule; the swap script (research/29) still
   helps FLR success between guest restarts within one boot.

## Sources

- ArchWorks (2026) RTX 50 series single-GPU pass-through guide (reset +
  iommu=pt + D3cold + firmware sections).
- CachyOS / Level1Techs Blackwell guide (RTX 5050/5060, dual-GPU host iGPU +
  dGPU VFIO, Looking Glass).
- PVE forum RTX 5060 Ti suspend/crash reports on ARL/Z890 (2026).
- szoran53/proxmox-kernel-6.17-gpu-passthrough (1:1 IOMMU patch context).
- research/02-gpu-passthrough.md (base setup, model sizing).

## Applied to / next steps

- Remove `iommu=pt` everywhere (plan/omalaptop.md A steps, docs/tasks.md A3,
  research/02 GRUB line) - done below in this session.
- Add `vfio-pci.disable_idle_d3=1` + udev D0 rule to Phase A kernel setup.
- Record "one VM run per host boot" as a GameWin11 usage rule in
  docs/omalaptop-notes.md and tasks.md Phase C.