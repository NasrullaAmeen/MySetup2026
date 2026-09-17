---
tags: [research, zfs, storage, snapshots, syncoid, arc, tuning]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# ZFS Deep-Dive & Tuning (1TB SK hynix + 256GB SN520)

Research date: 2026-09-17.

## Recommended layout

Two mismatched NVMe: do NOT stretch a mirror across them (pool capped by slowest/weakest; if either dies you lose everything anyway).

- 1TB rpool = root + VM/CT datastore, single-disk vdev (RAID0). Cheap to replace (reinstall + restore), ZFS still gives checksums, compression, snapshots, rollback.
- 256GB SN520 as plain `local` dir storage (ext4): ISOs, LXC templates (vztmpl), vzdump dump files, snippets.
- Mirror only if you buy a second 1TB NVMe; can `zpool attach rpool /dev/nvme0n1p3 /dev/nvme1n1p3` later without reinstall.
- Backups go off-box (PBS or USB/NAS), NOT on the same 1TB disk.

## Create commands

Best: install PVE with ZFS root on the 1TB via the installer. Manual equivalent:

```
zpool create -f -o ashift=12 -o autotrim=on \
  -O compression=zstd -O atime=off -O xattr=sa -O acltype=posixacl \
  -O mountpoint=none rpool /dev/disk/by-id/nvme-SK_hynix_...
```

- ashift=12 correct for modern 4K NVMe. 
- recordsize: containers use subvol datasets - set 16K on rpool/data; keep 128K/1M for a media dataset. VM zvols use pool blocksize (PVE GUI default 16K).
- Compression: zstd (default -3). Since OpenZFS 2.2 zstd runs an LZ4 fast-abort heuristic so incompressible media costs ~nothing.

```
zfs create -o recordsize=16k -o compression=zstd rpool/data
pvesm add zfspool local-zfs --pool rpool/data --content rootdir,images --sparse 1
```

## PVE specifics

- Installer creates rpool/ROOT/pve-1 (root), rpool/data (VM/CT images), rpool/swap.
- LXC rootfs/subvols appear as subvol-<id>-disk-0 under /rpool/data/; bind-mount extra datasets into a CT via mp0:.
- arc_max: PVE >=8.1 writes 10% of RAM clamped to 16GiB into /etc/modprobe.d/zfs.conf for fresh installs. On 62GB set your own cap:

```
echo "options zfs zfs_arc_max=12884901888" > /etc/modprobe.d/zfs.conf   # 12GiB
echo 12884901888 > /sys/module/zfs/parameters/zfs_arc_max              # live, no reboot
```

## Snapshots & replication

- GUI snapshots / `zfs snapshot rpool/data/subvol-100-disk-0@snap1` are near-free and instant, but live on the same disk.
- Scheduled retention: sanoid (keep hourly/daily/weekly; prune via `sanoid --cron`). Replication: syncoid `syncoid -r rpool/data remote:backup/rpool/data` (incremental zfs send|recv, resumable, compress).
- vzdump/PBS = real off-box backups with verification/dedup; snapshots = quick recovery. Grid: snapshots + syncoid daily, vzdump-to-PBS weekly.

## Scrubs & health

- Default cron: second Sunday monthly 00:24. Fine for NVMe. `zpool scrub rpool`, `zpool status -v`.
- `zpool set autoreplace=on rpool`. SMART: `smartctl -a /dev/nvme0n1` (consumer NVMe often lacks/varies telemetry; watch nvmereallocated/temperature).

## Tuning table

| Knob | Setting | Why |
|------|---------|-----|
| zfs_arc_max | 12GiB | ~20% of 62GB, single-disk box |
| ashift | 12 | 4K NVMe alignment |
| compression | zstd / lz4 | both abort on incompressible |
| atime | off | cuts metadata writes |
| autotrim | on | NVMe GC |
| autoreplace | on | self-heal single-disk |
| dedup | OFF | RAM tax ~5GB/TB, no benefit |
| recordsize | 16K data, 1M media | CT + media workloads |

## Migration from local-lvm/ext4

No supported in-place conversion of the root pool; qm migrate / pvesr cannot go lvmthin -> zfspool. RECOMMENDED: fresh PVE install on ZFS (the 1TB currently has no data), rebuild CTs, restore VMs via vzdump/PBS. Do it now while the disk is new - trivial once everything is in vzdump/PBS backups. Only wait if you're about to buy a second 1TB for the mirror.

## Sources

- https://pve.proxmox.com/wiki/ZFS_on_Linux
- https://pve.proxmox.com/pve-docs/local-zfs-plain.html
- https://pve.proxmox.com/wiki/Storage:_ZFS
- https://github.com/jimsalterjrs/sanoid
- https://forum.proxmox.com/threads/arc.161839