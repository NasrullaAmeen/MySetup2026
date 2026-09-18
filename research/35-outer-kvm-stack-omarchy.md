---
tags: [research, omalaptop, kvm, libvirt, virt-manager, omarchy, host, qemu]
created: 2026-09-18 17:00:45 +05
modified: 2026-09-18 18:25:03 +05
---

# Outer KVM host stack on Omarchy (Arch under the hood)

Part of the 2026-09-18 pivot: OmaLaptop host = Omarchy desktop (iGPU) that also
runs virt-manager/QEMU/KVM for the OUTER VMs (GameWin11, MainWin11) and the
nested ProxDev PVE VM. This note studies the host-side stack.

## Omarchy host facts (2026, from omarchy.org + omacom/omarchy-iso)

- Arch Linux base, Hyprland + Quickshell, systemd. Packages via pacman +
  AUR + the Omarchy repo. CLI: `omarchy update` (config+pacman -Syu),
  `omarchy pkg add <pkg>`.
- The Omarchy ISO is the supported install path. It mandates:
  - LUKS full-disk encryption (mandatory for Omarchy),
  - btrfs with compression,
  - Limine bootloader.
  So our plan/omalaptop.md Phase A must expect LUKS + btrfs + Limine on
  nvme0n1, NOT plain ext4. Kernel cmdline lives in Limine entries
  (`boot/limine.conf`), not GRUB.
- The ISO ships cloud-init `cidata` (NoCloud label) support; a sample PVE
  `qm create` for an Omarchy guest uses `q35 --bios ovmf --cpu host
  --ostype l26 --scsihw virtio-scsi-single` with EC/serial - a good XML
  template pattern for our outer VMs.
- AUR is available for anything missing (e.g. looking-glass, passthrough
  helper scripts).

## KVM/QEMU/virt-manager packages (Arch)

From ArchWiki "QEMU" + "Libvirt" + community guides (ArchWorks 2026):

```
sudo pacman -S qemu-desktop            # or qemu-full (more archs/features)
sudo pacman -S libvirt virt-manager edk2-ovmf swtpm dnsmasq
sudo pacman -S iptables-nft            # libvirt default NAT needs it
sudo pacman -S tpm2-tools              # swtpm guest TPM for Win11
```

- `qemu-desktop` vs `qemu-full`: desktop builds include the display
  backends we need; full adds extra targets/chips. Either works.
- Service: `sudo systemctl enable --now libvirtd virtlogd`
  (plus `virtnetworkd` pulled by libvirtd if the default network is used).
- Add user to `libvirt` group so virt-manager can talk to qemu:///system.
- kvm_intel nested (needed for the nested PVE VM):
  `/etc/modprobe.d/kvm_intel.conf` -> `options kvm_intel nested=1`
  (then `modprobe -r kvm_intel && modprobe kvm_intel` or reboot; verify
  `/sys/module/kvm_intel/parameters/nested` = Y).

## Networking model (per plan/networking.md + plan/omalaptop.md)

- Host owns Wi-Fi (and optionally USB-C hub LAN). Never bridge the wlan.
- libvirt NAT bridge `virbr0` is the default but we use our own host bridge
  `vmbr0` on 10.20.0.0/24 (gw 10.20.0.1) with netfilter masquerade to Wi-Fi
  (plan/networking.md runbook, research/33 for Wi-Fi reliability).
- Nested PVE guest gets 10.20.0.30 on vmbr0; inside PVE it builds its own
  vmbr 10.40.0.0/24 -> double-NAT (research/34).
- iptables-nft is required by libvirt for firewall rules; the host may also
  use nftables directly - pick ONE management tool to avoid rule clashes.

## libvirt hook pattern for passthrough VMs

ArchWorks-style hook dispatcher (official libvirt hooks + custom script):

- `/etc/libvirt/hooks/qemu` dispatcher (chmod 755) routing by
  `$1/$2/$3` = domain / start|stopped|prepare|release / phase.
- Per-VM config: `/etc/libvirt/hooks/qemu.d/<domain>/config` holding
  `GPU_BDF=0000:01:00.0`, `AUD_BDF=0000:01:00.1`, `VM_CPUS="0-3,8-11"`,
  `HOST_CPUS="4-7,12-15"`, paths, etc.
- start/stop scripts: modprobe vfio, cgroup cpuset isolation of host work,
  drop caches, set dynamic 2M hugepages, then virsh nodedev-detach/reattach.
- With a laptop dual-GPU split (iGPU = host, dGPU = always VFIO) we do NOT
  need the single-GPU claim/release dance - the dGPU belongs to vfio-pci from
  boot, so hooks only matter for: CPU pinning, hugepages, suspend gating,
  and the GameWin11<->MainWin11 swap.

## VM XML essentials (outer VMs)

- Machine: `q35`, firmware OVMF (`edk2-ovmf`), `pcie-root-port` slots for
  the hostdev and controllers (with q35 each device needs a pcie root port;
  virt-manager assigns automatically).
- CPU: `host-passthrough` (or `host-model`). For Windows add Hyper-V
  enlightenments so the guest clocks and drivers behave:
  `hv_relaxed hv_vapic hv_time hv_spinlocks hv_vpindex hv_runtime hv_synic
  hv_stimer hv_apic hv_reset hv_vpindex hv_frequencies`. MBEC/`-hypervisor`
  hiding is only needed by anti-cheat or WSL2 VBS - decide per game.
- Win11: swtpm2 + vTPM (`<tpm><backend type='emulator' version='2.0'>`),
  Secure Boot via OVMF. Plan Phase C step 9 (research/29).
- Disable `<pm>` suspend in VM XML so libvirt never asks a passed-through
  GPU machine to S3 (research/39 later: laptop suspend with GPU VM running
  is fragile - gate/suspend the domain first). This matches plan Phase C11.
- virtio-blk multi-queue + `<iothread>` and a `queue` size > 1 on the NIC
  for the nested PVE VM and GameWin11 (lowers IO latency a lot).

## Performance tuning (laptop budget, 24C/24T)

- CPU pinning: `vcpupin` + `emulatorpin` so the nested PVE VM (8 vCPU) and
  GameWin11 (8 vCPU) each pin to physical cores; keep 4-8 host cores free
  for the desktop. iothread pins keep virtio from bouncing cores.
- Hugepages: 2M dynamic hugepages via hook for the GPU VM; keep the desktop
  on regular THP (avoid 1G reclaim churn on a laptop).
- RAM: 62 GB usable. Budget table in plan/omalaptop.md: 16G nested PVE (fixed,
  no balloon), 16G GameWin11, 8G MainWin11, rest for desktop.

## Sources

- https://omarchy.org/manual/ - Omarchy manual (Arch, Hyprland, pacman).
- https://github.com/omacom/omarchy-iso - ISO + cidata cloud-init + sample
  PVE qm create line.
- ArchWiki: QEMU, Libvirt, PCI passthrough via OVMF.
- ArchWorks guide (2026) single-GPU pass-through setup + hooks + tuning.

## Applied to / next steps

- Update plan/omalaptop.md Phase A with LUKS+btrfs+Limine reality, qemu-desktop
  package set, systemd units, iptables-nft, and the hook config layout.
- Build the vmbr0 NAT runbook on the Omarchy host first (Phase A step 4).
- Test hook CPU-pinning with the throwaway VM before GameWin11.