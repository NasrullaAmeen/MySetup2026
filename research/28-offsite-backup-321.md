---
tags: [research, backup, 3-2-1, offsite, pbs, rclone, restic, omalaptop, proxlab]
created: 2026-09-18 10:15:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 28. 3-2-1 backup across OmaLaptop + ProxLab (offsite)

Research date: 2026-09-18. This repo's docs live on GitHub; the *data* on the
two Proxmox hosts (VM disks, NAS media, configs) is not yet covered by
offsite backups. This note turns the existing local PBS plan
([05-backup-pbs.md](./05-backup-pbs.md)) into a working 3-2-1:

- **3 copies**: primary data + PBS local + offsite.
- **2 media**: local disk + remote object storage (or remote PBS).
- **1 offsite**: cloud or a far-away friend's box.

## Current state

| Host | Local protection | Offsite | Risk |
|---|---|---|---|
| OmaLaptop (laptop) | PNY "Hdscythe" 1T via USB (detached), vzdump/PBS plan not yet built | none | Laptop theft/short-circuit = all VMs gone |
| ProxLab (X99) | (planned) PBS + ZFS snapshots | none | Same |
| Git docs (this repo) | GitHub remote | GitHub | OK - already offsite |

## Net scheme

```
primary (host NVMe/SATA)
   |  vzdump / PBS encrypt
   v
PBS datastore (ProxLab or PNY USB)   <- local copy #2
   |  rclone / restic --encrypt
   v
offsite object storage (B2 / R2 / rsync.net)  <- copy #3, encrypted
```

1. **Local PBS** - one Proxmox Backup Server (best as an LXC or a tiny VM on
   ProxLab or on the PNY-attached host) storing both hosts' backups:
   dedup + encryption + retention. Schedule: daily at 02:00, keep 7d/4w/6m.
2. **Detached copy** - keep the PNY "Hdscythe" USB: take a monthly
   `syncoid`/`zfs send` of the PBS datastore (or `vzdump` archives) onto it,
   then physically detach. This is the "burglar-proof" copy.
3. **Offsite** - encrypt in PBS (`prune` + encryption) and/or push with:
   - `rclone` / `restic` of the PBS datastore to Backblaze B2 (~$6/TB/mo) -
     simplest for the ~2 TB we have.
   - or `pbs`'s native "sync" to a second PBS reached over WireGuard/Tailscale
     (a friend's server, a VPS). More moving parts, no egress cost if already
     paid.
   Both options dedup-encrypt client-side; never upload plaintext.

## Config (beyond VM data) gets offsite too

- `/etc/pve` (cluster/config) snapshot:
  `tar czf /root/pve-config-$(date +%F).tar.gz /etc/pve` + push to git/object.
- Datacenter-level: **pvesh keys** for API tokens already in Bitwarden;
  replicate the note (offline vault export) - the GitHub repo is not the only
  key to this kingdom.
- NAS media (`zfs send nas` to offsite is usually too big -> back up the *catalogs*, not the bytes: arr databases, Plex/Jellyfin libraries, family photos only).

## Offsite targets compared (2026 pricing ballparks)

| Target | Cost/TB/mo | Egress | Notes |
|---|---|---|---|
| Backblaze B2 | ~$6 | ~$10/TB (cheap) | native rclone, no minimum |
| Cloudflare R2 | ~$15 | free egress | great for many small pulls |
| rsync.net | ~$30 | n/a | ZFS-native, you can `zfs send` directly |
| Friend's PBS over Tailscale | ~0 | LAN VPN | needs a counterpart box |

For ~2-3 TB scheduled daily incremental, **B2 with rclone** is the right
default; switch to R2 if the VM images get served/restored frequently.

## Recovery drills (do not skip)

- Quarterly: restore one VM from PBS into a scratch VM (ProxLab cheap cycles).
- Semi-annual: restore the same from the detached PNY copy (boot-test).
- Prove the offsite restore path once (download a small datastore, boot it).

## Open questions

- Who pays egress if media grows (evergreen: media does grow). Keep offsite
  scope = VM/system data + catalogs + photos, NOT the full NAS media pool.
- Where PBS runs: LXC on ProxLab (preferred - it is the always-on quieter
  box) vs the PNY USB host. Decide with [26-*](./26-proxlab-storage-nas.md)
  storage build.