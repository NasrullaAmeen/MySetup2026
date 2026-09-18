---
tags: [research, homelab, proxmox, architecture, best-practices]
created: 2026-09-17 21:24:01 +05
modified: 2026-09-18 18:25:03 +05
---

# Proxmox Homelab Architecture and Best Practices

Research date: 2026-09-17. Target: OmaLaptop (PVE 9.2.20, Core Ultra 9 275HX 24C/24T, 62 GB RAM, RTX 5060, 2x NVMe, greenfield).

## VM vs LXC per service

| Workload | Choice | Notes |
|----------|--------|-------|
| Pi-hole / AdGuard / Uptime Kuma | LXC (unprivileged) | near-native perf, ~50 MB RAM idle, 2-5 s boot |
| Postgres / MariaDB | LXC | standalone DBs |
| Home Assistant OS | VM | appliance, give 2 cores / 4 GB |
| OPNsense / pfSense | VM | BSD + NIC passthrough |
| Windows | VM | - |
| Docker host (docker-compose) | VM | one Ubuntu/Debian VM for all stacks - dominant 2026 pattern |
| Jellyfin / Plex w/ GPU transcode | VM | RTX 5060 passthrough |
| Internet-facing / untrusted | VM | best isolation |

- PVE 8.2+/9.1 add native OCI containers (run many Docker images at LXC overhead), but Docker-in-LXC breaks on upgrades - use VM for Docker.
- 2026 split: ~60% LXC / 25% OCI / 15% VM. CPU type default `x86-64-v2-AES`; use `host` only if never migrating.

## Node / cluster

- Single node is the right start. HA formally needs 3 nodes for quorum (2 + QDevice is fragile).
- Cluster = tight Corosync + pmxcfs; cannot join nodes that already run guests.
- Proxmox Datacenter Manager 1.x: loose API-token coupling, live migration across hosts - NOT a cluster substitute. Skip with one node.
- Pattern: 1 capable node + independent PBS + tested restores.

## PVE 9.x features (timeline)

| Version | Notable |
|---------|---------|
| 9.0 (2025-08) | Debian 13 Trixie, kernel 6.14, QEMU 10.0, LXC 6.0, ZFS 2.3.3, live RAIDZ add |
| 9.1 | kernel 6.17, OCI, external metric server via OpenTelemetry |
| 9.2 (2026-05) | kernel 7.0, QEMU 11.0, LXC 7.0, ZFS 2.4, SDN WireGuard+BGP, custom CPU models |

## CPU / memory / disk tuning

- ZFS ARC cap: PVE 8.1+ fresh installs cap at 10% RAM (16 GB max). Set explicitly:
  ```
  /etc/modprobe.d/zfs.conf:  options zfs zfs_arc_max=8589934592  (8-16 GB)
  update-initramfs -u
  ```
- dwappiness=10; avoid ballooning on ZFS-backed VMs.
- Pools from `/dev/disk/by-id/`, ashift=12, compression=lz4, recordsize 16K (VM zvols) / 1M (media-backup). Skip L2ARC/SLOG on all-NVMe.
- vCPU overcommit ~2:1 light services, 1:1 encode/LLM. Leave cores for the host.
- Arrow Lake idle power: PVE 9.2 kernel 7.0 reaches C10 (~42 W) - fixed in 6.17+. Kernel 7.0 alone may boost idle; add `pcie_aspm.policy=powersave`, enable C-states + ASPM in BIOS.

## IaC patterns

- Terraform + `bpg/proxmox` provider (not Telmate) + dedicated API token (never root).
- Ansible `community.proxmox` collection; add `wait_for_connection` (cloud-init 30-120 s).
- Cloud-init template once: `qm set 9000 --ide2 local-lvm:cloudinit --serial0 socket --vga serial0 --agent enabled=1 && qm template 9000`.
- Store ISO/cloud images on `local`, LXC templates in `local:vztmpl`.
- Worth it >5 VMs; skip for 2.

## Pitfalls to avoid

1. No backups / PBS on same host (see backup note).
2. Wrong storage at install: plain LVM kills snapshots/backup mode - choose ZFS or LVM-thin.
3. Enterprise repo without subscription = no updates - switch to no-subscription.
4. No QEMU guest agent in VMs.
5. WiFi bridged into vmbr - does not work.
6. Not backing up /etc/pve config.
7. Same NVMe for PVE + PBS datastore.
8. Interface rename risk on kernel upgrades - pin names (proxmox-network-interface-pinning).

## Sources

- https://pve.proxmox.com/wiki/Roadmap
- https://www.proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-2
- https://forum.proxmox.com/threads/intel_idle-c-states-in-proxmox-with-intel-series-200-cpus.179143/
- https://wz-it.com/en/knowledge/proxmox/what-is-proxmox-datacenter-manager/