---
tags: [research, docker, proxmox, lxc, portainer, compose, oci]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Docker on Proxmox: Patterns (LXC vs VM)

Research date: 2026-09-17.

## Consensus (2026)

- VMs: PVE's official recommendation. Own kernel, tested upgrades, live migration, HA, cleaner isolation. Cost: ~512MB-1GB RAM + 5-15% overhead. Proxmox does NOT support/test Docker-in-LXC; it can break at major upgrades (cgroup v1->v2, AppArmor, runc syscalls).
- Unprivileged LXC (nesting=1,keyctl=1): community standard for single-node homelabs. ~100MB overhead, near-native perf (5-13% faster than KVM), 3s boots, per-CT backups. Risk = compatibility drift on PVE kernel/tooling updates.
- Rule: efficiency/density -> LXC; security, uptime, GPU, internet-facing, or "just works" -> one VM hosting all Docker. Never install Docker on the PVE host.

## Decision table

| Criterion | LXC + Docker | VM + Docker |
|-----------|-------------|-------------|
| RAM overhead | ~100MB | ~512MB+ |
| Perf (CPU/IO) | ~0-1% overhead | 5-15% slower |
| Setup difficulty | High (nesting/apparmor quirks) | Low |
| Upgrade safety | Can break on PVE updates | Supported |
| Live migration | No | Yes |
| Backup | Per-CT, bind mounts excluded | Whole-VM vzdump |
| Security | Shared kernel, weaker | Strong (VT-x) |
| GPU/USB | fiddly (privileged or /dev/dri mount) | PCI passthrough |

## Path A: Docker in unprivileged Debian LXC

```
pct create 101 local:vztmpl/debian-12-standard_12.2-1_amd64.tar.zst \
  --hostname docker01 --memory 4096 --cores 4 --rootfs local-lvm:32 \
  --net0 name=eth0,bridge=vmbr0,ip=10.10.10.101/24,gw=10.10.10.1 \
  --features nesting=1,keyctl=1 --unprivileged 1 --ostype debian --start 1
# inside CT:
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker
docker info | grep -i "storage driver"   # expect overlay2
```

- WARNING: bridge-nf-call-iptables is disabled - harmless; ignore or run daemon with `--iptables=false` if only compose bridge nets.
- If `vfs` instead of overlay2: add `lxc.apparmor.profile: unconfined`; keep --storage-driver=overlay2 default (works on ZFS with kernel >=6.2 d_type).
- Bind-mounts from host need subuid mapping (lxc.idmap) or chmod - else permission denied on volumes.

## Path B: Docker in Debian/Ubuntu VM

```
qm create 201 --name docker-vm --memory 8192 --cores 4 \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:80 --ide2 local:iso/ubuntu-24.04.iso,media=cdrom \
  --boot order=scsi0 --agent 1 --cloudinit-user local:cloudinituser \
  --ipconfig0 ip=10.10.10.201/24,gw=10.10.10.1
```

Pros: qm snapshots + vzdump (snapshot mode + guest agent = fs-consistent), live migration, HA, no nesting quirks. Size disk generously (images + builds).

## Management

- CLI: zero overhead, learn it first.
- Dockge (~30-50MB): compose-focused, stores stacks as plain compose.yaml on disk (/opt/stacks) - no lock-in; best default for single-host homelab.
- Portainer (~100-200MB): full image/volume/network UI, RBAC, multi-host agents; risk = stacks stored in its DB - use Git integration or keep compose files on disk.
- Tailscale sidecar pattern exposes apps at *.ts.net https with zero published ports - recommended over exposing 9443.

## Compose patterns

- Bind mounts for configs/media; named volumes for DBs; .env per stack; restart: unless-stopped; custom bridge networks (default bridge has no DNS); internal: true networks for DBs behind only the reverse proxy.

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    restart: unless-stopped
    env_file: .env
    volumes: [vw-data:/data]
    networks: [internal]
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports: ["80:80","443:443"]
    volumes: [./Caddyfile:/etc/caddy/Caddyfile:ro, caddy-data:/data]
    networks: [internal, external]
networks: { internal: {internal: true}, external: {} }
volumes: { vw-data: {}, caddy-data: {} }
```

## Backups

- Whole-VM vzdump = simplest; DBs need app-consistent dumps (pg_dump/mysqldump) or agent-frozen snapshot. LXC: only root disk by default - tick Backup on extra mountpoints; bind mounts NEVER backed up.
- Docker-side: per-volume tar (`docker run --rm -v vol:/source:ro -v $(pwd):/backup alpine tar czf /backup/vol.tgz /source`); compose+.env in git; restic for dedup+offsite. Test restores quarterly.

## OCI in PVE 9.1

Tech preview: pull OCI images in GUI, converted to LXC rootfs (no docker daemon). No compose/orchestration, no idempotent restart, squashed layers (no dedup), upgrade = recreate CT (data disks destroyed - keep config disk or use mountpoints). Use for lightweight tinker; not a Docker replacement yet.

## Sources

- https://pve.proxmox.com/pve-docs/chapter-vzdump.html
- https://forum.proxmox.com/threads/oci-images-in-lxc-release-9-1.176273
- https://blog.gntech.me/posts/2026-05-17-docker-proxmox-lxc-guide/
- https://smallgrid.uk/guides/run-docker-proxmox-vm-or-lxc/
- https://forum.proxmox.com/threads/lxc-and-docker-compatibility-now.185224