---
tags: [research, pbs, backup, lxc, proxlab, dedup, encryption, retention]
created: 2026-09-18 11:00:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 32. Running Proxmox Backup Server (PBS) as an LXC on ProxLab

Research date: 2026-09-18. Strategy is in [05-backup-pbs.md](./05-backup-pbs.md)
and the 3-2-1 net in [28-offsite-backup-321.md](./28-offsite-backup-321.md).
Every Proxmox note ends at "run PBS"; this one is the concrete "do it on
ProxLab" runbook (client = OmaLaptop + ProxLab). Synergy with
[26-proxlab-storage-nas.md](./26-proxlab-storage-nas.md)? The PNY "Hdscythe"
drive is our detached target - keep PBS *itself* on a disk that stays
attached, use the PNY only for monthly `syncoid`/`vzdump` copies.

## Layout on ProxLab

| Piece | Where | Notes |
|---|---|---|
| PBS service | LXC (Debian 13, unprivileged, 2c/4G) | separate from NAS/media LXC |
| PBS datastore | WD SSD or its own dir on ZFS mirror | dedup needs random IO; do NOT put the datastore on the slow mirror |
| Datastore type | `cachedir` (filesystem datastore) | fine for homelab GSG; skip ZFS-datastore |
| PNY USB | monthly `zfs send`/`vzdump` staging | detached backups (burglar-proof) |

## Install (LXC path, 2026 method)

1. Create the LXC on ProxLab: unprivileged Debian 13, authkey-based,
   fixed IP in 10.30.0.0/24 (e.g. .10). Give it 2 cpu / 4G / 16-32G rootfs
   on the pool.
2. In the LXC add the datastore dir:
   ```
   apt install proxmox-backup-server
   proxmox-backup-manager datastore create sdd-store /srv/pbs --gc-schedule "daily,02:00"
   proxmox-backup-manager create-remote <snapshots of the datastore> ...
   ```
   (datastore must be on a real fs - mount the WD/ZFS path into the container.)
3. Web UI `https://<lcx-ip>:8007`; create a backup user + API token
   (e.g. `backup@pam`/token) - give the token read/write on the datastore.
4. On OmaLaptop + ProxLab: PBS client = the host itself (`proxmox-backup-client`
   on the PVE ISO/host): add `Prune` props in the UI, set a retention
   (keep 7d/4w/12m), enable **encryption** (client-side, via a `.enc`
   password file, NOT stored anywhere on the box).

## Client-side schedule (OmaLaptop + ProxLab)

- Proxmox-backup-client job (host) - VM/CT backup:
  ```
  proxmox-backup-client backup root.pxar:/ ... --repository 10.30.0.10:8007:store
  ```
  Better: use **PVE's built-in GUI -> Datacenter -> Backup** with a
  `job` pointing at `store` (SMB/NFS not needed; PVE speaks PBS natively).
  Add to cron/systemd:
  - 02:00 daily (VMs), 02:30 (configs `/etc/pve`), verify monthly.
- Enable **Verify + GC** schedule on the datastore (GC daily low-traffic
  hours; verify weekly rotated).
- Failures to ntfy/Telegram - add the hook later (research/22).

## Retention and space math (PBS dedup reality)

- PBS dedups **server-side per host** - so cross-host dedup baseline is
  small if both hosts run similar VMs; the real win is *not re-sending
  unchanged blocks* at 02:00.
- 2 hosts x ~4 VMs avg 20-80 GB each: first full ~250-400 GB; daily deltas
  tiny. With verify/prune, ~2-3x that for retention copied offsite is
  realistic. The 1 TB PNY mirror only holds ~2-3 months at that size;
  better: keep 4-6 weekly zfs sends on the PNY, not the 30-day daily set.

## Encryption keys & offsite

- Put the **PBS encryption passphrase** in Bitwarden (no plaintext on the
  hosts, no `pw` files readable). Offsite (28) receives the encrypted store.
- Do a **restore drill** quarterly (28): boot a scratch VM from the
  datastore, then from the PNY copy.

## Gotchas

- `proxmox-backup-server` in an unprivileged LXC: the container needs
  `mpX: /store,mp=/srv/pbs` mountpoint + enough disk quota for dedup
  metadata (16G rootfs + datastore on mp).
- Do NOT let the PBS disk fill: GC/verify need headroom; freeze
  (feature-frozen) alerts >85% - dedup re-winds otherwise.
- PVE backup jobs to PBS use port 8007 TCP; firewall rules must allow
  OmaLaptop <-> ProxLab only for this port (research/04 + plan/proxlab.md).

## Open questions

- PBS as LXC on ProxLab vs a dedicated 4th VM: LXC is right unless NVMe
  passthrough of the whole datastore disk is desired - LXC cannot hotplug
  raw NVMe easily; a VM could. Keep LXC unless we outgrow 2 TB.
- Cross-host dedup: both hosts share a PBS server, so dedup aggregate
  applies; verify the deltas really shrink (they should for daily VMs).