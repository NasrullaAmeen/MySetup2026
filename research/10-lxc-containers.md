---
tags: [research, proxmox, lxc, containers, pct]
created: 2026-09-17 21:34:13 +05
modified: 2026-09-17 21:42:45 +05
---

# LXC Containers on Proxmox

Research date: 2026-09-17.

## LXC vs Docker

- LXC on PVE = system container: full init (systemd), shares host kernel, near-native I/O, 30-80MB idle RAM, boots in 1-2s, native ZFS snapshots and PBS backups.
- Docker = app container; on PVE run it nested in an LXC (`features: nesting=1,keyctl=1`; overlay2 needs OpenZFS 2.2+/PVE 8.1+) or in a VM.
- Official docs: Docker should run in a QEMU VM for isolation/live-migration. Community: Docker-in-LXC is fine for trusted internal workloads.
- PVE 9.1 adds OCI-image support for LXC (system containers GA-ish, app containers tech preview; no Compose).
- Pitfalls: systemd needs nesting; Docker-LXC needs AppArmor `lxc.apparmor.profile: unconfined` (mqueue deny bug on PVE9/Debian 13); runc 1.2+ on unprivileged LXC can hit `ip_unprivileged_port_start`; FUSE mounts freeze during snapshot backups.

## Privileged vs unprivileged

- Unprivileged is the GUI default: root->UID 100000 via /etc/subuid/subgid; a container escape lands as an unprivileged host user.
- Bind mounts then need host ownership matched to the mapped range (100000+) or `lxc.idmap` + subuid entries to pass a UID through.
- Privileged maps container root = host root; escape = full host compromise (real CVE-2022-0185). Use only with a documented reason and keep single-purpose, unexposed. [DANGER] Never expose privileged containers to the network edge.

## pct cheat sheet

```
pveam update && pveam available --section system
pveam download local debian-12-standard_12.12-1_amd64.tar.zst
pct create 100 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname vaultwarden --unprivileged 1 \
  --cores 2 --memory 2048 --swap 512 \
  --rootfs local-lvm:16 --storage local-lvm \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.50/24,gw=192.168.1.1,firewall=1 \
  --mp0 local-lvm:8,mp=/var/lib/vaultwarden,backup=1 \
  --features nesting=0 --onboot 1 --startup order=2 \
  --timezone host --start
pct set 100 --features nesting=1,keyctl=1 --swap 1024 --tty 1
pct snapshot 100 pre-upgrade && pct exec 100 -- apt update && pct exec 100 -- apt upgrade -y
pct rollback 100 pre-upgrade
pct restore 100 /var/lib/vz/dump/vzdump-lxc-...tar.zst
pct enter 100 ; pct pull/push 100 <src> <dst>
```

Notes: `--features fuse=1,mknod=1` for FUSE/devices; onboot + `--startup order=` for boot ordering; tty limits consoles.

## Networking

- Default is a veth pair on a Linux bridge (vmbr0): guest gets a real LAN IP, so LXC has no NAT by default; services reachable directly.
- Static IP: `ip=192.168.1.50/24,gw=...`.
- `firewall=1` on net0 enables the Proxmox per-VE firewall (/etc/pve/firewall/100.fw).
- KNOWN ISSUES: Docker macvlan inside LXC breaks when PVE MAC filtering/firewall is on (disable MAC filter or use publish port mapping); PVE firewall on an interface mangles ARP/MTU in some configs. Debug order: interface UP -> bridge/VLAN/PVID -> ARP -> firewall.

## Storage / backup

- rootfs on local-lvm (thin, ext4 raw) or ZFS subvol (snapshot-friendly).
- Extra mp0 = managed volume (backup/snapshot able, `backup=1`) or bind mount from host.
- Bind mounts are EXCLUDED from vzdump ("not a volume") - back them up separately or convert to volumes/NFS.
- `pct restore` reuses config (simple mode) or overrides mp/rootfs (advanced mode).
- PVE 9 quirk: LXC backup fails when an extra mountpoint is on a ZFS zvol ("mount point not empty") - empty the rootfs mountpoint dir; move mp to file-based storage. autofs on /mnt breaks backup (can't create /mnt/vzsnap0).

## Apps & GPU in LXC

- Vaultwarden, AdGuard, Pi-hole: bare unprivileged LXC works great.
- Pi-hole/AdGuard need static IP, port 53/80/443 handling.
- Docker-in-LXC (docker-ce): nesting=1,keyctl=1, expect iptables bridge-nf-call warning and AppArmor issues.
- Ollama + GPU: host runs the NVIDIA driver; container gets cgroup2 device rules + bind-mounted /dev/nvidia* + matching user-space libs (--no-kernel-modules); for Docker-in-LXC set no-cgroups = true in nvidia-container-toolkit; host/container driver versions must match.

## Best practices

- Static IPs + unique hostnames (PVE rewrites /etc/hosts).
- `--timezone host`; onboot with `--startup order=`.
- Snapshot before every apt upgrade; update via `pct exec <ct> -- apt ...`.
- Swap per CT; test restore regularly.

## Service placement table

| Service | Recommendation |
|---------|----------------|
| Vaultwarden, Pi-hole, AdGuard | bare unprivileged LXC |
| Jellyfin/Plex (GPU/USB), Frigate, Ollama | bare LXC + device passthrough |
| nginx/Caddy reverse proxy, Tailscale, DNS | bare LXC (~50MB each) |
| Immich, Nextcloud, Grafana+Prometheus | Docker-in-VM (compose) |
| Docker host for 10+ microservices | 1 medium LXC (4C/8G) or VM |
| Untrusted / multi-tenant / needs live-migration | Docker-in-VM |

## Sources

- https://pve.proxmox.com/wiki/Linux_Container
- https://pve.proxmox.com/wiki/Unprivileged_LXC_containers
- https://pve.proxmox.com/pve-docs/pct.1.html
- https://github.com/community-scripts/ProxmoxVE
- https://blog.gntech.me/posts/2026-05-17-docker-proxmox-lxc-guide/