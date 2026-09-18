---
tags: [research, windows, gaming, gpu, passthrough, vtpm, secure-boot, omalaptop, proxlab]
created: 2026-09-18 11:00:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 29. Windows 11 + gaming VMs on Proxmox (GameWin11 / MainWin11)

Research date: 2026-09-18. Two Windows guests appear in the plans:
- **OmaLaptop GameWin11**: RTX 5060 dGPU passthrough (gaming, 8c/16G).
- **OmaLaptop MainWin11**: iGPU (8086:7d67) shared, swapped with MainArch.
- **ProxLab desktop VM**: RX 5700 XT passthrough (research/25).

This note is the Windows-side companion to [02-*](./02-gpu-passthrough.md)
(NVIDIA) and [25-*](./25-amd-rx5700xt-passthrough.md) (AMD). Grab the
driver/kernel-side steps there; here: guest config, Win11 requirements,
anti-cheat reality, and audio.

## Win11 hard requirement: TPM 2.0 + Secure Boot

- PVE 9 provides **vTPM 2.0** (swtpm) and OVMF Secure Boot for QEMU.
- Win11 setup demands vTPM + SB on, else setup refuses / later breaks
  (BitLocker, OneDrive, VBS/Core isolation).
- Build with `machine: q35`, firmware **OVMF** (*not* SeaBIOS),
  `tpmstate0` backed by a small disk (RAM-backed tmpfs unsupported), and
  Secure Boot enabled. PVE GUI can add TPM + resize. BitLocker then works.
- Windows 11 since 2024-25 also asks for a Microsoft Account unless you
  skip with the OOBE `bypassnro` trick. Keep that in the runbook.

## The passthrough stack (guest wiring)

| Guest | GPU | Host | Extra |
|---|---|---|---|
| GameWin11 | RTX 5060 (10de:2d59 + 22eb audio) | OmaLaptop | `cpu: host,hidden=1,aes=1`, OVMF |
| MainWin11 | iGPU 8086:7d67 | OmaLaptop | exactly one of MainArch/MainWin11 runs |
| ProxLab desktop | RX 5700 XT (731f + ab38) | ProxLab | vendor-reset (25), PLX subtree (24) |

Common gotchas:
- **hypervisor.hidden** + `hv-vendor-id` random: hides KVM signature from
  NVIDIA/AMD guests; code-43 prevention.
- Drive: use **VirtIO SCSI/block** for disks (fast); virtio-net + `netkvm`
  drivers from the virtio-win ISO. `fstrim`/`discard` = `SSD emulation off`
  unless testing.
- **Audio**: pass the GPU's HDMI/DP audio with the GPU (RTX: 10de:22eb;
  Navi: 1002:ab38), or use a separate USB audio device. Scream/Spice audio
  for thin clients only.
- vGPU titan RTX-style splits are NOT available for consumer cards; no
  actual iGPU/dGPU sharing while a guest holds it.

## HAGS / gaming features

- Hypervisor-accelerated GPU scheduling: enable **HAGS** in Win11
  Settings->Display->Graphics->Default graphics settings; GPU accel on OVMF
  guests works on recent virtio/gpu only when real GPU is passed; with a
  real GPU it is just a Windows toggle.
- **ReBar**: enable on the card (AMD/NVIDIA) + `x-vga=on` is NVIDIA-only;
  for AMD keep ReBar off unless PVE already maps it.
- Vsync/stutter is largely gone on passthrough; the two big real-world
  problems remain *reset on guest restart* (see 25) and *frame pacing of
  audio sync* over HDMI audio.

## Anti-cheat: the honest matrix

| Game/launcher | Passthrough result |
|---|---|
| Steam, most SP titles | work |
| EAC / BattlEye games | usually work with `hv-vendor-id` set; occasionally ban on kernel-level identity |
| Valorant (Vanguard), games w/ HSM checks | frequently blocked - Vanguard especially is hostile to VMs |
| Fenix/ubisoft (R6) | mixed |

- Kernel-level anti-cheat reads SMBIOS/ACPI. PVE cannot fake everything; if
  a specific title refuses to launch, it is the game, not the config.
- Practical stance: keep a **booting Windows partition option** (bare-metal
  boot entry alongside PVE - see [31-*](./31-workstation-to-pve.md)) as the
  fallback for online FPS that demand bare metal.

## Guest savings / ops

- Snapshot the Windows volume **before any big update/driver install**
  (VirtIO/q35 snapshot-capable).
- Disk: give Windows a dedicated disk (not share with Linux); NTFS over
  VirtIO block + QEMU fstrim works.
- Power: enable `win11`-inline time sync; disable Windows Fast Startup
  (it half-shuts-down and breaks passthrough reboots).
- On both hosts, set GameWin11/desktop VM **start = manual** (dGPU slot is
  one-at-a-time); MainWin11 = the iGPU swap (omalaptop.md hard rules).

## Open questions

- Which anti-cheat titles the household plays (drives whether bare-metal
  fallback matters).
- ReBar on the Gigabyte RX 5700 XT for ProxLab - needs PVE 8.2+/9 mapping.