---
tags: [research, homelab, proxmox, storage, zfs, lvm]
created: 2026-09-17 21:24:01 +05
modified: 2026-09-18 18:25:03 +05
---

# Storage - ZFS vs LVM-thin for Proxmox Homelab

Research date: 2026-09-17. Target: 2x NVMe (256 GB + 1 TB), 62 GB RAM, PVE 9.x.

## Feature comparison

| Feature | ZFS | LVM-thin (local-lvm) |
|---------|-----|----------------------|
| RAID at install | mirror/raidz | no |
| Snapshots | instant, near-free | fast, COW overhead with many |
| Checksums / bit-rot detect | yes (fletcher4/blake3) | no |
| Self-healing | yes (mirror/raidz) | no |
| Compression | lz4/zstd inline | no |
| Scrub/verify | periodic scrub | none |
| Native replication | zfs send/recv, syncoid | none |
| Thin provisioning | sparse zvols | yes |
| RAM footprint | ARC (cap it) | minimal |
| Failure mode | slow, safe | pool 100% -> writes fail/data loss |

Consensus 2025-26: choose ZFS unless RAM-poor.

## ARC sizing (62 GB RAM)

PVE 8.1+ caps fresh-install ARC at 10% RAM / 16 GB max. Manual pools default 50% (62.5% since ZFS 2.3). Set explicitly:

```
# /etc/modprobe.d/zfs.conf
options zfs zfs_arc_max=17179869184   # 16 GiB
options zfs zfs_arc_min=8589934592    # 8 GiB
update-initramfs -u && reboot
```

Best homelab ARC at 64 GB RAM: 12-16 GiB. Verify via `arc_summary` / `cat /proc/spl/kstat/zfs/arcstats` (c_max).

## NVMe specifics

```
zpool create -o ashift=12 -O compression=lz4 -O atime=off vmpool /dev/disk/by-id/nvme-...
zfs create -o recordsize=16K vmpool/vms      # VM zvols (default volblocksize 16K since ZFS 2.2)
zfs create -o recordsize=1M -o compression=zstd vmpool/backups
```

- ashift=12 for consumer NVMe (4K sectors); 13 only for verified 8K enterprise drives. Immutable after create.
- TRIM: PVE cron runs monthly `zpool trim`; optional `zpool set autotrim=on`.
- Skip L2ARC/SLOG on all-NVMe (low latency already).

## Single disk vs mirror (this box)

Mirror of 256 GB + 1 TB = 256 GB usable + runs at slowest disk - wasteful.

| Layout | Result |
|--------|--------|
| 1 TB single-disk ZFS pool (root + VMs) | ~900 GB usable - recommended |
| 256 GB separate single-disk pool | ISOs/templates/backups |

Single-disk ZFS detects corruption (checksums) but cannot self-heal - backups beat redundancy.

## Migration LVM-thin <-> ZFS

Cannot convert root install disk in place. Options: add disks and create a ZFS pool (add to storage.cfg), or reinstall with ZFS. VM move between pools: offline only (GUI Move Volume); `qm migrate` fails cross-format. Enable sparse on ZFS storage (`pvesm set local-zfs --sparse 1`). Safest: vzdump backup -> fresh ZFS install -> restore.

## Backup interplay

- vzdump snapshot mode works on both; wants thin + discard storage (sparse ZFS or lvmthin).
- PBS works with either; with ZFS you can point PBS at ZFS snapshots (`--snapshots yes`).
- Native `zfs send -i` exists only on ZFS - key DR advantage (pve-zsync/syncoid).
- Known PVE 9 bug (2025-12): LXC backup fails when extra mount point is on a zvol - keep CT disks file-based or suspend mode.

## PVE 9 gotchas

- Keep user-space ZFS and kernel module aligned (2.4 userspace + 2.3 kmod broke `zpool scrub`).
- ZFS 2.3.4 snapshot-access kernel panic fixed in 2.4.0.
- Keep pools < 80% full; no SLOG needed on NVMe; consumer NVMe without PLP = sync-write durability risk (acceptable homelab).

## Recommendation for OmaLaptop

ZFS installed on the 1 TB NVMe (root + VM zvols). 256 GB as second single-disk pool for ISOs/templates. ashift=12, lz4, atime=off, 16K volblocksize, ARC cap 16 GiB, monthly scrub + trim, PBS (separate box) with snapshots.

## Sources

- https://forum.proxmox.com/threads/lvm-thin-vs-zfs-for-local-vm-storage-on-a-single-node-pve-pros-cons-in-2026.184660/
- https://cr0x.net/en/proxmox-zfs-vs-lvm-thin-benchmark/
- https://klarasystems.com/articles/arc-and-l2arc-sizing-for-proxmox/
- https://blog.gntech.me/posts/2026-05-26-zfs-pool-design-proxmox-homelab/
- https://techfuelhq.com/tutorials/zfs-proxmox-pool-setup-arc-tuning-2026/