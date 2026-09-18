---
tags: [research, omalaptop, nested, proxmox, kvm, ops, performance]
created: 2026-09-18 17:00:45 +05
modified: 2026-09-18 18:25:03 +05
---

# Nested Proxmox VE: operations and performance (running PVE in a VM)

Part of the 2026-09-18 pivot: the "ProxDev" is a nested PVE VM
(8c/16G, host-passthrough CPU) living on the Omarchy/libvirt host. This note
details what works, the expected performance, and the operational rules.

## Supported, and prerequisites

- Proxmox VE officially supports running inside a VM (PVE wiki "Nested
  Virtualization"): treat the physical host as level-0, the PVE VM as
  level-1, and PVE's own guests as level-2.
- Level-0 (Omarchy host) requirements:
  - CPU must expose vmx to the PVE guest -> guest CPU type `host` /
    `host-passthrough` at the libvirt level (plan hard rule);
  - kvm module must have `nested=1` (Arch: `/etc/modprobe.d/kvm_intel.conf`
    -> `options kvm_intel nested=1`; verify
    `/sys/module/kvm_intel/parameters/nested` = Y);
  - `VMCS shadowing` (Haswell +, Arrow Lake has it) makes depth-2 VMs fast.
- Sanity inside the PVE guest: `cat /proc/cpuinfo | grep vmx`,
  `ls -l /dev/kvm`, run a tiny test VM there.
- If `nested=0` or CPU not host-passthrough, PVE cannot start VMs with KVM -
  it falls back to TCG (`qm start` fails without KVM support) - unusably
  slow. Do NOT attempt.

## Performance expectations (PVE wiki measurements)

| Metric | Effect of depth-1 nesting |
|---|---|
| Inner VM boot time | slower (~20-30 s vs a few s) |
| Disk seq read | ~400-600 MB/s vs ~2000 MB/s (vhost) |
| Random IO / network | small overhead (virtio multi-queue closes it) |
| Idle CPU overhead | ~10-25% of one vCPU |
| RAM overhead | negligible |
| Latency-sensitive guests | avoid depth-2 gaming/RT |

- I/O-heavy depth-2 workloads are the worst case - keep them small. Hosting
  the lab LXC + small software VMs is exactly right for a nested PVE.
- Give the nested PVE VM virtio-blk multi-queue + `iothread` and a virtio NIC
  with `queues > 1` at the outer level (research/35).

## Inside the nested PVE: filesystem choice

- Do NOT use ZFS for nested PVE storage ("PVE wiki: ZFS in a virtual
  environment" caveats; false stats, RAM hungry, poor perf without direct
  disk access). Use ext4/xfs on LVM-thin or a plain dir/raw on qcow2.
- Our plan: single thin qcow2 (~80 G) on the hynix pool (research/34). The
  guest just sees a vda virtio disk - all normal PVE tools work on top.
- Enable discard to let qcow2 thin reclaim; run fstrim on schedule.

## Hard limits of the nested tier (plan hard rules)

- NO GPU passthrough and NO re-pass-through inside: the PVE VM cannot hand a
  PCI device onward to its own guests. No NVMe passthrough, no USB/IPMI
  passthrough either (they are not physical here). Lab tier = software only.
- No live migration between nodes (standalone instance by design, research/27
  vs cluster notes).
- Nested PVE sees virtual appliances, so no S.M.A.R.T./nvme-cli on the host
  disks from inside - run those on the Omarchy host (research/30).

## Operations that matter

- Time: keep the PVE guest on the host clock. In libvirt XML use
  `<clock offset='utc'><timer name='rtc' tickpolicy='catchup'/>...`);
  inside PVE run chrony so drift after host S3 resumes is corrected fast.
- Reboot after host suspend (S3): the nested guest state can be mangled -
  the plan Phase C11 test asserts "nested PVE reboot after resume"; wrap it
  in a script/alias (virsh start/suspend/resume + `virsh reboot` guest).
- Snapshots: qcow2 internal snapshots via `virsh snapshot-create-as` give a
  fast safety catch even when PVE is down; plus PVE-level vzdump/PBS for VM
  level backups (research/32). Keep host-hypervisor snapshots and PVE
  vzdump in separate roles - do not mix.
- Resource: fixed 16 G no balloon (research/34 keeps nested stable); grow
  cores/RAM on demand, not by default.
- LXC nesting inside nested PVE: `features: nesting=1` still applies
  (research/10); docker-in-LXC needs OpenZFS or overlay-friendly storage.

## Sources

- https://pve.proxmox.com/wiki/Nested_Virtualization - official nested guide
  and the performance table.

## Applied to / next steps

- Phase B runbook in plan/omalaptop.md already matches; add the ext4/LVM-thin
  fs decision and the clock/snapshot notes into the B steps.
- Confirm `nested=1` on Omarchy right after install (Phase A step 2).