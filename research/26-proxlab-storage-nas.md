---
tags: [research, proxlab, storage, zfs, lvm, nas, sata, hdd]
created: 2026-09-18 10:15:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 26. ProxLab storage + NAS design (3x SATA + NVMe)

Research date: 2026-09-18. Target: **ProxLab** X99 box. It has no iGPU and a
lot of drives; this note decides where PVE, VM disks, and bulk data live, and
how to expose a NAS share. Drives (from [../plan/proxlab.md](../plan/proxlab.md)):

| Drive | Size | Type | Speed tier |
|---|---|---|---|
| nvme0n1 Lexar NM710 | 500G | NVMe Gen4 (PCIe 3.0 on this board) | fast |
| sda WD Blue WDS100T2B0B | 1T | SATA SSD | mid |
| sdb Seagate ST1000LM035 | 1T | 2.5" 5400 rpm HDD | slow |
| sdc Seagate ST1000LM049 | 1T | 2.5" 7200 rpm HDD | slow |
| sdd PNY 1T (USB, "Hdscythe") | 1T | SATA SSD via USB | backup |

## Design

| Tier | Drive | Use | Layout |
|---|---|---|---|
| System | Lexar NVMe 500G | Proxmox VE root + ISO + pvebackup staging | plain ext4/btrfs, ~100G |
| VM pool | WD SATA SSD 1T | VM/CT disks (fast, cached) | **LVM-thin** (cheap snapshots) or ZFS |
| Bulk/NAS | sdb + sdc (2 HDDs) | media, backups-2nd, long-term files | **ZFS mirror** (mirror-0) |
| Backup | PNY USB 1T | PBS datastore / vzdump target | single disk, detached between runs |

## Why not other combinations

- **RaidZ1 across all 3 SATA**: unbalanced (SSD vs 5400/7200 HDDs), hostage to
  the slowest disk, and WD SSD wears from ZFS writes. No.
- **Mirror SSD + HDD**: mirror speed = slowest (HDD). No.
- **Single ZFS pool of 3 mixed disks**: same problem + single-disk failure in
  a non-redundant config kills the pool.
- **Passthrough the C610 SATA controller to a NAS VM**: you lose host access
  to those disks and the AHCI controller reset-on-reboot pain outweighs the
  zoning benefit. Keep disks on the host ZFS and share out via SMB/NFS.

## Recommended storage build (ProxLab)

1. Install PVE on the Lexar NVMe (single root disk, ~100G used).
2. `pvesm` add the **WD SSD as LVM-thin** (`local-wdthin`) for all VM/CT
   disks - matches the OmaLaptop hynix LVM-thin pattern.
3. ZFS mirror: `zpool create nas mirror sdb sdc` with `ashift=12,
   recordsize=1M` (media), compression `lz4`. Datasets:
   - `nas/media` (movies/show/music - target for the *arr stack)
   - `nas/data` (documents), `nas/backup2` (cold VM backups)
4. Do NOT make the PNY USB part of any pool; attach it only for PBS runs or
   `rsync` mirrors, then detach (matches "detached backup" on OmaLaptop).
5. Share the NAS:
   - NFS for Linux VMs/LXC: `zfs set sharenfs=on nas` (or export in
     `/etc/exports`).
   - SMB for Windows/media clients: `samba` in an LXC or the host;
     reference [../research/20-media-nas-stack.md](../research/20-media-nas-stack.md).

## ZFS notes specific to this platform

- 64-109G RAM: plenty for ARC, but it is a dual-socket server - cap the ARC
  at ~32G so VMs keep their memory (`/etc/modprobe.d/zfs.conf`,
  `options zfs zfs_arc_max=34359738368`).
- Set `zfs_commit_timeout` / sync behavior for media (aflushd). `recordsize
  =1M` + `xattr=sa` help SMB shares.
- Slow HDDs: keep snapshots short (a few days), use `syncoid` to the PNY for
  long-term, and consider `autotrim=on` only on the SSD (trim on HDD pool is
  a no-op).
- HDDs: 5400 rpm becomes the bottleneck for anything > 40 MB/s sequential; if
  the two HDDs prove too slow for the NAS goal, re-purpose them as pure
  backup mirrors and run media off the WD SSD.

## NAS reader/review

- Media stack: [20-media-nas-stack.md](./20-media-nas-stack.md).
- Services in LXC: [10-lxc-containers.md](./10-lxc-containers.md),
  [09-homelab-services.md](./09-homelab-services.md).

## Open questions

- ZFS mirror of two laptop HDDs is fine for "I want my photos safe", not for
  IO: confirm the media users actually stream from sdb/sdc and not from the
  WD SSD before committing to the tier split.
- Whether to also export the PNY (Hdscythe) via the *same* LXC for cross-host
  access - currently a OmaLaptop-only backup target; prefer keeping it USB-only.