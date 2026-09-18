---
tags: [research, proxlab, vfio, gpu, passthrough, amd, navi10, rx5700xt]
created: 2026-09-18 10:15:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 25. AMD Radeon RX 5700 XT (Navi 10) VFIO passthrough

Research date: 2026-09-18. Target: **ProxLab** - the single disk-based X99
host must run headless and hand its only GPU (RX 5700 XT) to a desktop VM.
This is very different from OmaLaptop's RTX 5060 story ([02-*](./02-gpu-passthrough.md)).
Navi 10 has a famous **reset bug**; the fix is the community `vendor-reset`
module.

## The Navi reset bug in one paragraph

AMD cards before Navi 2x often cannot reset themselves (no working FLR/BACO)
after a VM exits. Result: `qm stop` then `qm start` fails with
`error writing '1' to /sys/bus/pci/devices/0000:00:00.0/reset: Inappropriate
ioctl`, QEMU `pci_irq_handler` assertion, "no PCI device found", or a stuck
D3 state - until you reboot the *host*. The only reliable reset path is a
firmware-internal reset driven by `vendor-reset`, which hooks
`pci_dev_specific_reset` via ftrace and implements the AMD-specific
PSP/Mode1/BACO sequences for Vega/Navi.

## Proven 2026 recipe (from live homelab reports)

1. **vBIOS / hardware**: keep the GPU's own vBIOS ROM in the VM config
   (`romfile=` on the host) only if guest drivers misbehave; start without it.
2. **Kernel args** (PVE 9, kernel ~7.x):
   ```
   iommu=pt pcie_aspm=off pci=noaer
   ```
   - `intel_iommu=on` / `amd_iommu=on` are **defaults since 6.8** - omit.
   - `video=efifb:off` / `nofb nomodeset` are obsolete; for a headless host
     use `initcall_blacklist=sysfb_init` instead (loses console output
     though - debug with a second PC).
3. **vendor-reset via DKMS** (required - "the only working reset method on
   Navi10", BACO reset):
   ```
   apt install proxmox-headers-$(uname -r)
   git clone https://github.com/gnif/vendor-reset.git /usr/src/vendor-reset
   ```
   - On kernels **>= 6.12/6.14** vendor-reset does not compile out of the
     box; apply the upstream fixes (github.com/gnif/vendor-reset PR #103 /
     #104): line 32 of `src/amd/amdgpu/atom.c` must use `linux/unaligned.h`
     instead of `asm/unaligned.h`.
   - Build/install:
     ```
     cd /usr/src/vendor-reset && dkms add . && dkms build vendor-reset/0.1.1 && dkms install vendor-reset/0.1.1
     echo "vendor-reset" >> /etc/modules                # load EARLY
     update-initramfs -u && reboot
     modprobe vendor-reset
     ```
   - Re-run DKMS after **every** `apt full-upgrade` kernel bump.
4. **Enable device-specific resets** once loaded, before starting the VM:
   ```
   echo 'device_specific' > /sys/bus/pci/devices/0000:04:00.0/reset_method
   ```
   (Persist via a systemd oneshot or udev rule; verify in dmesg that the
   module's NV_NAVI10 messages appear.)
5. **Bind to vfio-pci early** (host bind, not just runtime):
   `options vfio-pci ids=1002:731f,1002:ab38` (GPU + HDMI audio) so they never
   grab amdgpu/snd_hda_intel on boot.
6. **Pass the whole subtree** - this box has a PLX switch (see 24-*): attach
   bridge 03:00.0 + GPU 04:00.0 + audio 04:00.1. Bridge and device are in
   separate IOMMU groups (83 / 84 / 85) so this is a normal 3-attach VM,
   no ACS patch needed.
7. **D3cold stuck-GPU fix** (WZ-IT / VFIO 2026 guides): after `qm stop` the
   GPU can enter D3cold; add a qm **hookscript** (`pre-start`/`post-stop`)
   that clears the power state / re-binds (`echo 1 > .../reset` after
   unbind-remove-rescan) - the r/VFIO "stable RX 5700 XT passthrough on PVE 9"
   walkthrough bundles this with a watchdog and a PBS-backup-no-suspend fix.

## Guest side

- Windows: `machine: q35`, OVMF (UEFI), `cpu: host,hidden=1,aes=1`. First boot
  installs MS Basic Display, then the AMD driver (avoid Radeon Adrenalin
  "Windows update" path quirks - install full Adrenalin).
- Linux guest: current Mesa/amdgpu is fine; enable `x-vga=on` (PrimaryGPU) is
  *NVIDIA-specific, leave off* for AMD.
- Glitchy console during guest boot is normal with AMD GTT; use a toggled
  output via `x-vga` only if the motherboard has a second display path.

## When vendor-reset still fails

- Re-verify module load: `modprobe -v vendor-reset`, `dkms status`, dmesg for
  `NV_NAVI10` / `vendor_reset_hook: installed`.
- `rmmod` leftovers: `dkms remove vendor-reset/0.1.1 --all` then reinstall.
- If resets stay flaky, last-resort: reboot the host between GPU VM cycles
  (the "no fix whatsoever" caveat some report). Budget that into ProxLab
  availability.

## Verdict for ProxLab

- GPU passthrough is achievable and, with vendor-reset + hookscripts,
  *restart-stable* on PVE 9 (many 2026 reports confirm). Expect a day of
  fighting D3/reset edge cases the first time; then it's set-and-forget.
- Because the box has no iGPU, the GPU cannot stay on the host - accept that
  ProxLab is headless (like OmaLaptop) and admin it over the web UI/SSH.