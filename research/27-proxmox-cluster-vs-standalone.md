---
tags: [research, proxlab, omalaptop, cluster, ha, quorum, qdevice, replication]
created: 2026-09-18 10:15:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 27. Proxmox cluster vs 2x standalone (OmaLaptop + ProxLab)

Research date: 2026-09-18. Now that ProxLab (X99) joins OmaLaptop (laptop),
the "old" default is a **2-node Proxmox cluster with a QDevice** for HA.
This note works out what we actually get, what it costs, and the caveats
(2026 PVE 9.x).

## What clustering does and does not buy

| Feature | 2x standalone | 2-node cluster + QDevice |
|---|---|---|
| One pane of glass UI | no | yes |
| Live migration between hosts | no (offline relocate only) | yes (needs shared storage or ZFS replication) |
| HA restart on node failure | no | yes (watchdog-fenced, needs repl/shared storage) |
| pmxcfs sync of configs | no | yes |
| Quorum failure mode | none | if both nodes + QDevice unreachable, read-only |
| Corosync traffic | none | must be stable, isolated VLAN |

## The 2-node quorum trap

- 2 nodes = 2 votes; losing one node leaves the survivor at 1/2 - **no
  quorum** -> pmxcfs goes read-only, no VM/CT start or config writes. HA
  cannot act. This is exactly the outage the second server was bought for.
- Fix: an external **QDevice** (**corosync-qnetd** on a 3rd tiny host,
  **corosync-qdevice** on both nodes). Expected votes becomes 3, quorum = 2,
  so one node + the arbiter stays quorate.

## QDevice placement for this homelab

- The arbitrator votes *differently* than the cluster - it should survive the
  same failures the nodes share. Best candidates:
  1. The home router (10.10.10.1) if it can run a tiny container (small RAM,
     256M, one daemon).
  2. A Raspberry Pi / spare SBC - most common homelab choice.
  3. A *third* tiny Linux box (old laptop) - cleaner than a Pi, but for two
     nodes the practical one is the router/Pi.
- Install on ONE arbiter and both nodes:
  ```
  # on arbiter (Debian/Ubuntu-like):
  apt install corosync-qnetd
  # on each PVE node:
  apt install corosync-qdevice
  # from one node:
  pvecm qdevice setup <arbiter-ip>
  ```
  `pvecm status` should say `Quorate Qdevice`, Expected votes 3.

## HA - do not hand-wave

- Requires watchdog fencing (PVE self-fences the pair: `softdog` by default,
  `iTCO_wdt` hardware watchdog is available on the X99/C610 board - select in
  `/etc/default/pve-ha-manager`).
- Requires data the survivor can run: **ZFS replication (`pvesr`)** since we
  have no Ceph/shared storage. Async, default 15 min (min 1 min) - failover
  loses the last delta. Only replicate what tolerates ~RPO 15 min (e.g. the
  MainArch/MainWin11 tier and key LXC), never the NAS media pool.
- **cloud-init disk on local-zfs** breaks `qm migrate` with broken-pipe;
  delete `ide2` (the cloudinit drive), migrate, then re-add it. There is no
  GUI dialog for this.
- PVE 9: HA Groups are deprecated -> use **node-affinity rules**
  (`ha-manager` / GUI "Node Affinity").

## Recommendation for THIS setup

- **Start standalone.** Both hosts are petting-zoo single-purpose boxes
  (laptop + workstation) that mostly run always-on local services; none of
  the "lost node" scenarios are worth corosync + fencing + replication
  complexity yet.
- **Cluster only if** a real workload needs auto-restart (e.g. MainArch must
  be available 24/7 no matter what). Then: QDevice on the router/Pi,
  `pvesr` jobs for the tier that matters, node-affinity rules, and a hard
  "pull the plug" failover test before relying on it.
- Keep cross-host **PBS** as the real safety net (dedup + history beats
  replication); see [05-backup-pbs.md](./05-backup-pbs.md) and
  [28-offsite-backup-321.md](./28-offsite-backup-321.md).

## Sources / further reading

- Proxmox COROSYNC/QDevice docs; WZ-IT 2026 2-node QDevice guide; ProxmoxR
  quorum explainer; TechFuelHQ 2-node/3-node HA 2026; NetCollege ZFS
  replication + HA + cloud-init gotchas (all 2026).

## Open questions

- Where the arbiter physically lives (router vs Pi vs old laptop) once decided.
- Whether MainArch's RPO 15 min window is acceptable - probably yes for a
  daily-driver desktop, no for anything transactional.