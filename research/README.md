---
tags: [research, index, homelab, proxmox]
created: 2026-09-17 21:21:06 +05
modified: 2026-09-17 21:38:22 +05
---

# Research Index

Deep-research notes on homelab / Proxmox topics. Each note has its own tags and timestamps per [../AGENTS.md](../AGENTS.md).

## Notes (core)

- [01-homelab-architecture.md](./01-homelab-architecture.md) - Proxmox homelab architecture and best practices.
- [02-gpu-passthrough.md](./02-gpu-passthrough.md) - GPU passthrough with RTX 50-series for AI/LLM.
- [03-storage-zfs-vs-lvm.md](./03-storage-zfs-vs-lvm.md) - ZFS vs LVM-thin storage decisions.
- [04-networking.md](./04-networking.md) - VLANs, firewall, Tailscale for homelab.
- [05-backup-pbs.md](./05-backup-pbs.md) - Backup strategy with Proxmox Backup Server.
- [06-security-hardening.md](./06-security-hardening.md) - Proxmox security hardening.
- [07-monitoring.md](./07-monitoring.md) - Monitoring with pve-exporter, Grafana, Prometheus.

## Notes (platform / ops)

- [08-proxmox-cli.md](./08-proxmox-cli.md) - Proxmox CLI cheatsheet (qm, pct, pvesh, pveum, pveam, pvesm).
- [10-lxc-containers.md](./10-lxc-containers.md) - LXC containers: pct workflow, unprivileged, networking, backup.
- [13-iac-templates.md](./13-iac-templates.md) - Terraform/OpenTofu (bpg/proxmox), Ansible, cloud-init templates.
- [14-proxmox-api.md](./14-proxmox-api.md) - Proxmox REST API automation (curl/pvesh/tokens).
- [19-zfs-deep-dive.md](./19-zfs-deep-dive.md) - ZFS layout, tuning, snapshots, syncoid, migration.
- [23-docker-patterns.md](./23-docker-patterns.md) - Docker on Proxmox: LXC vs VM, Portainer/Dockge, OCI.

## Notes (apps / services)

- [09-homelab-services.md](./09-homelab-services.md) - Popular homelab services catalog with deploy advice.
- [11-community-scripts.md](./11-community-scripts.md) - community-scripts.org helper scripts (install, safety).
- [12-ai-stack.md](./12-ai-stack.md) - Ollama + Open WebUI on RTX 5060 8GB, models, tuning.
- [20-media-nas-stack.md](./20-media-nas-stack.md) - *arr media stack + gluetun + SMB/NFS sharing.

## Notes (network / access)

- [16-tailscale.md](./16-tailscale.md) - Tailscale subnet routers, serve, ACLs for remote access.
- [17-reverse-proxy-tls.md](./17-reverse-proxy-tls.md) - Caddy/Traefik/NPM, ACME, Authelia auth.
- [18-virtual-router-opnsense.md](./18-virtual-router-opnsense.md) - OPNsense VM, VLAN router-on-a-stick, NIC pairing.

## Notes (security / reliability)

- [15-power-efficiency.md](./15-power-efficiency.md) - Arrow Lake 275HX idle power, C-states, ASPM tuning.
- [21-tfa-sso.md](./21-tfa-sso.md) - PVE TFA, Authelia/Authentik SSO, SSH 2FA.
- [22-monitoring-stack.md](./22-monitoring-stack.md) - Prometheus + pve-exporter + Grafana + Alertmanager/ntfy.

## Applied to prodev

- prodev: Intel Core Ultra 9 275HX, 62.2 GB RAM, RTX 5060 Max-Q, 2x NVMe. See [../docs/prodev-hardware.md](../docs/prodev-hardware.md).
- Research date: 2026-09-17.