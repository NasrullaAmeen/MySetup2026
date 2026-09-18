---
tags: [research, storage, nvme, wearout, smartctl, monitoring, omalaptop, replacement]
created: 2026-09-18 11:00:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 30. NVMe health & wearout on the hypervisors (OmaLaptop 96%/99%)

Research date: 2026-09-18. OmaLaptop's drives are already near end-of-life:
- `/dev/nvme1n1` SK hynix P41 1 TB: wearout **99%** (VM pool!).
- `/dev/nvme0n1` WDC SN520 256 GB: wearout **96%** (PVE OS).

These are laptop drives that were used hard before becoming a hypervisor.
Plan = use them now, watch them, and replace before failure - not when
`/dev/nvme0n1` vanishes mid-backup.

## Metrics that actually matter

Consumer NVMe often lack full vendor telemetry; the *reliable* endurance
signal for these is the **NAND wear** percentage (smartctl `Percentage Used`
for the hynix/Intel/SanDisk class; WD exposes `Media Wearout Indicator`).
Ignore raw bytes-written spikes (some report garbage), watch:

- `Percentage Used` / `Media Wearout %` (SMART attr 245/177).
- `Media Errors` + `Number of Error Information Log Entries` (grows on death).
- Temperature - the biggest preventable killer on laptops.
- `Unsafe Shutdowns`.

`smartctl -a /dev/nvme0n1` + `sudo smartctl -a /dev/nvme...` (run as root;
SMART on NVMe needs `-d nvme` implicitly).

## Monitor it like load

- Tie into the existing stack ([07-monitoring.md](./07-monitoring.md) had
  smartctl_exporter for the client): add `smartctl_exporter` + a Grafana
  panel with `percentage used` and temp per host and a **>90% wearout
  alert**.
- No monitoring yet? A cron one-liner is better than nothing:
  ```
  0 7 * * * smartctl -a /dev/nvme0n1 | awk '/Percentage Used/{print}' | mail -s "OmaLaptop wearout" root
  ```
- Snapshot `/etc/pve` to git + Backup: wearout alert = start the replacement
  plan (not the panic button).

## What the worn % means

- Wearout is *estimated NAND endurance consumed*, not a prediction of the
  drive dying today - a 99%/96% pair can run months more at light
  hypervisor write loads. But **at 100% the vendor no longer honors the
  warranty and some firmware shift to write-through + slowdown**.
- Hypervisor WAF (write amplification from LVM-thin snapshots / ZFS) is what
  burns P41 fast. Cut it:
  - LVM-thin: enable `discard`/`fstrim` on guests, keep snapshot churn low.
  - ZFS: `recordsize` alignment, `compress=lz4`, `autotrim=on` (only where
    TRIM is supported), avoid heavy `sync=always`.
  - Logs/metrics go to VM disks, not the pool.

## Replacement plan (budget-ready)

| Role | Current | Replace with | Cost sanity |
|---|---|---|---|
| OmaLaptop VM pool | hynix P41 1 TB (99%) | 1-2 TB Gen4 NVMe (SN850X class) | ~$90-130 |
| OmaLaptop OS | WD SN520 256 GB (96%) | 512 GB Gen4 (or reuse pool slot) | ~$40 |
| ProxLab OS | Lexar NM710 500 GB (new) | n/a now | - |

Recommended sequence for the pair (zero-downtime-ish):
1. Buy one 1-2 TB replacement; `clonezilla`/`dd` the P41 -> new pool while
   the box is briefly down, or add new disk as a 4th pool member and
   migrate VMs via `pvesm move`. LVM-thin: `pvesm move` each VM disk.
2. Confirm `smartctl` on the new drive shows 0%.
3. Keep the P41 as a hot spare / offline backup drive (it still boots at
   99% for a while - label it "wearout 99%").
4. Replace the WD OS disk only when the P41 slot is stable.

- Read the *current* wear first (fast):
  `watch -n 0 smartctl -a /dev/nvme{0,1}n1` after any big VM disk move.

## Open questions

- Whether to go ZFS-vdev-instead-of-replace cheaper: a second 1 TB in
  mirror costs more than a single bigger drive; given both disks are ~100%
  worn, spare capacity (2 TB single) is the better buy.
- If wearout hits 100% on the VM pool first: are we OK with read-only for a
  day while the swap happens? (Probably - non-destructive VM drain is known.)