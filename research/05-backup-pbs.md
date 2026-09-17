---
tags: [research, homelab, proxmox, backup, pbs, disaster-recovery]
created: 2026-09-17 21:24:01 +05
modified: 2026-09-17 21:42:45 +05
---

# Backup Strategy - Proxmox Backup Server and 3-2-1 for Homelab

Research date: 2026-09-17.

## Backup options compared

| Option | Dedup | Incremental | Portable | Best for |
|--------|-------|-------------|----------|----------|
| vzdump to storage | no | no (full each run) | yes (.vma.zst/.tar.zst) | one-off / small |
| Proxmox Backup Server (PBS) | chunk dedup 3-10x | incremental-forever | no (server-managed) | regular automated backups |
| restic / borg / kopia | yes | yes | yes | app data + host OS, files not VMs |

PBS: AES-256-GCM client-side encryption, live-restore, file-level restore in GUI, verify/prune/GC/sync jobs. Officially recommended for regular automated backups. Do NOT rclone/rsync/restic into a PBS datastore (breaks it).

## PBS placement

| Placement | Verdict |
|-----------|---------|
| Bare-metal on separate hardware (old box: 2-4 cores, 4 GB RAM + ~1 GB per TiB) | BEST |
| PBS VM/CT on a 2nd/3rd PVE node, or storage on NAS | acceptable |
| PBS VM on the SAME host | interim only - NOT a real backup of the host; must pair with offsite copy |

Officially: "Installing the backup server directly on the hypervisor is not recommended."

## PBS features

- Dedup at chunk level across VMs and over time; incremental-forever.
- Client-side encryption (encryption-key AES-256-GCM, master-pubkey recovery).
- Datastore sync jobs (pull-sync recommended for ransomware resistance; `--encrypted-only`).
- Verify jobs (daily new, then monthly re-verify-all via `--outdated-after 7`).
- Prune (keep-last/daily/weekly/monthly/yearly) + GC weekly.

## Recommended schedule (single host)

| Time | Job | Retention |
|------|-----|-----------|
| 02:30 nightly | PVE host config via proxmox-backup-client | 7 daily / 4 weekly / 3 monthly |
| 03:00 nightly | vzdump VMs+CTs (snapshot, zstd) to local PBS | 3 last / 7 daily / 4 weekly / 3 monthly |
| 04:00 nightly | datastore sync (encrypted) to remote PBS | 14 daily / 8 weekly / 12 monthly |
| daily | verify job | - |
| weekly | GC | - |
| monthly | test restore (qmrestore --unique) | - |

3-2-1: 3 copies (live + local PBS + offsite), 2 media types, 1 offsite. A backup never restored is not a backup.

## Backing up the PVE host itself

Do NOT vzdump the host root. Back up config: /etc/pve (pmxcfs: VM/CT configs, users, storage, jobs), /etc/network/interfaces, /etc/hostname, /etc/hosts, /etc/fstab, /etc/vzdump.conf, /etc/ssh, /root, /etc/lvm.

Via PBS (install proxmox-backup-client on PVE):

```
proxmox-backup-client backup root.pxar:/ --repository root@pbs@pbs.local:store
```

## Config examples

/etc/pve/storage.cfg (PBS storage):

```
pbs: pbs-backup
    datastore main
    server 192.168.1.50
    content backup
    username backup@pbs
    fingerprint XX:...
    encryption-key autogen
```

PBS CLI:

```
proxmox-backup-manager datastore create main /backup/main
proxmox-backup-manager verify-job create nightly-verify --store main --schedule "daily 04:00" --outdated-after 7
proxmox-backup-manager prune-job create --store main --keep-daily 7 --keep-weekly 4 --keep-monthly 3
proxmox-backup-manager garbage-collection start main
```

vzdump job (GUI or /etc/pve/jobs.cfg):

```
vzdump 123 --storage pbs-backup --mode snapshot --compress zstd \
  --prune-backups keep-last=3,keep-daily=7,keep-weekly=4,keep-monthly=3
```

## Restore

- Same node: GUI (PBS -> backup -> Restore) or `qmrestore <path> <vmid> --storage local-lvm --unique`, `pct restore`.
- New node: fresh PVE install, `pvesm add pbs ...`, backups reappear, restore VMs/CTs. Host itself: fresh install then restore config tarball before guests.
- Bandwidth: `pvesm set pbs-backup --bwlimit restore=102400`.

## Sources

- https://pve.proxmox.com/wiki/Backup_and_Restore
- https://pbs.proxmox.com/docs/maintenance.html
- https://pbs.proxmox.com/docs/installation.html
- https://github.com/barminetech/homelab-backup