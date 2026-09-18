---
tags: [research, nested, kvm, virt-manager, proxmox, qemu, omalaptop]
created: 2026-09-18 12:45:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 34. Nested Proxmox VE under KVM (OmaLaptop spends its GPU-free life)

Research date: 2026-09-18. Pivot confirmed by the user: OmaLaptop becomes an
**Omarchy desktop host** running virt-manager + QEMU/KVM, with a nested
**ProxDev PVE VM** inside it for the headless lab tier. Bare-metal
PVE is abandoned; the desktop and the "server" now share one laptop.

## The one hard rule

PCI passthrough only works at the OUTER libvirt level. A device assigned to
the nested PVE VM can NOT be re-assigned onward to that VM's guests. So:

- **GPU-bearing VMs (GameWin11 + RTX 5060, MainWin11) = outer virt-manager
  guests.** Never inside nested PVE.
- **Nested ProxDev PVE = lab tier:** LXC services (AdGuard, PBS, monitoring,
  Tailscale, *arr), small software-rendered VMs, snapshotting + backup jobs.
  No GPU anywhere inside.

## Why nested PVE at all

- Host stays a real desktop (Omarchy/Hyprland on iGPU) - travel = normal
  laptop, and the lab simply suspends.
- PVE gives battle-tested LXC, snapshots, vzdump/PBS, web UI for the lab
  without trusting the desktop's bleeding-edge desktop stack with it.
- Cost: nested virtualization overhead (small with `nested=1`) + a VM
  within a VM - acceptable for the lab tier.

## Enabling nested virt on the Omarchy host

- Host CPU: Arrow Lake (PHN16S-71). Check `/sys/module/kvm_intel/parameters/nested`.
  If `N`:
  ```
  echo 'options kvm_intel nested=1' > /etc/modprobe.d/kvm_intel.conf
  # or kernel cmdline: kvm-intel.nested=1
  ```
- Rebuild/restart libvirtd after; the NESTED VM must use CPU mode
  **host-passthrough** (or host-model with +vmx/+svm) or Windows inside
  nested VMs will not see acceleration.
- QEMU must not hotplug vCPU in a way that drops vmx; keep `cputune`
  static. Suspend/resume the laptop can mangle nested state - expect to
  reboot the nested guest after resume (test at Phase C).

## Nested PVE VM shape (locked)

| Item | Value |
|---|---|
| Name | `ProxDev` (guest OS = PVE 9.2 ISO) |
| Cores / RAM | 8 / 16G now; grow on demand via virt-manager |
| Disk | thin qcow2 on the hynix 1TB host pool (decision) |
| NIC | virtio on the host bridge (10.20.0.x) |
| CPU | host-passthrough + nested flags |
| Machine | q35 + OVMF (UEFI) |
| vTPM | swtpm (needed if it will host a Win VM - not planned) |

## Storage decision: thin qcow2 (why not NVMe passthrough)

Chosen: **thin virtio disk file** on the hynix pool, not VFIO whole-disk.
- Simpler: host keeps control (snapshots at libvirt level plus PVE level),
  no reserved whole API to the guest, shareable with outer VMs.
- Cost: tiny io overhead vs passthrough; hynix is the same 1TB either way.
- The nested PVE sees its block device as a VM disk; it can still do
  LVM-thin/ZFS inside that block if desired.

## Networking (double-NAT)

```
Wi-Fi / wired uplink  (host owns it)
  -> host NAT (iptables, 10.20.0.0/24)           [plan/networking.md]
     -> outer VMs incl. GameWin11, MainWin11     (10.20.0.x)
     -> nested ProxDev PVE guest                 (10.20.0.30 e.g.)
        -> its vmbr: 10.40.0.0/24 (PVE-internal)
           -> lab LXC + small VMs                (10.40.0.x)
```
Inner lab services reach out via PVE's own NAT to 10.20.0.1. Inbound to the
lab: host DNAT -> PVE guest -> PVE guest DNAT. Two layers of NAT - fine for
services, remember for mDNS/captive tests.

## What stays from the old plan

- Wi-Fi host ownership + NAT runbook (plan/networking.md) - unchanged.
- NVMe wearout watch (research/30) - now host-level (hynix 99%).
- PBS/backup strategy (research/05, 28, 32) - PBS can run as an LXC inside
  nested PVE backing up both worlds (owner: nested PVE).
- ProxLab plans (research/24-28) unaffected; nested PVE is standalone
  (no cluster, per user).

## Dropped from the old plan

- Bare-metal PVE: host is no longer PVE. Repo/enterprise/nag steps go away.
- MainOS-on-iGPU tier: the Omarchy host IS the daily desktop.
- MainWin11 on iGPU passthrough: iGPU is host-only; MainWin11 is an outer
  virtio-gpu VM (or a dGPU swap candidate with GameWin11 - one at a time).
- Cluster/HA with ProxLab (research/27): dropped for OmaLaptop.

## Reference

- PVE nested requirements: qemu must advertise vmx/svm (host-passthrough);
  `nested=1` on kvm module or `kvm-intel.nested=1`.
- libvirt: `<cpu mode='host-passthrough'/>`; swtpm for vTPM; OVMF pkg.
- All 2026-09-18 decisions live in plan/omalaptop.md (authoritative).