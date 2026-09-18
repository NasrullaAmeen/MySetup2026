---
tags: [tasks, todo, docs]
created: 2026-09-17 20:53:22 +05
modified: 2026-09-18 09:40:00 +05
---

# Tasks

## Merge (2026-09-18)

- [x] Merge plan docs (hardware/networking/proxdev/proxlab) into Setup repo under plan/ (see ../plan/README.md)

## Infrastructure

- [ ] Verify web UI "Shell" works as darko@pve after root disable
- [x] Disable root SSH login on host (PermitRootLogin no)
- [ ] Decide GPU passthrough / VM plan (RTX 5060 idle) - see research/02-gpu-passthrough.md (recommends LXC + Ollama)
- [ ] Storage: migrate local-lvm to ZFS? See research/03-storage-zfs-vs-lvm.md (recommends ZFS on 1 TB)
- [ ] Set up VLAN layout / segment home network - see research/04-networking.md
- [ ] Deploy PBS for backups (nightly vzdump + host config) - see research/05-backup-pbs.md
- [ ] Monitoring stack (pve-exporter + Grafana) in LXC - see research/07-monitoring.md

## Homelab (new)

- [ ] AdGuard Home / Pi-hole as first LXC (research/09-homelab-services.md, 10-lxc-containers.md; one-line install via research/11-community-scripts.md)
- [ ] Docker VM for self-hosted apps - decide LXC vs VM (research/23-docker-patterns.md)
- [ ] TFA (TOTP) for darko@pve (research/06-security-hardening.md, 21-tfa-sso.md)
- [ ] Reverse proxy + TLS (Caddy recommended; research/17-reverse-proxy-tls.md)
- [ ] Tailscale subnet router LXC for 10.10.10.0/24 (research/16-tailscale.md)
- [ ] Ollama + Open WebUI LXC with RTX 5060 pass-through (research/12-ai-stack.md)
- [ ] Media *arr stack in Docker (research/20-media-nas-stack.md)
- [ ] IaC provisioning - OpenTofu + bpg/proxmox + cloud-init templates (research/13-iac-templates.md)
- [ ] Cron reporting via REST API token (research/14-proxmox-api.md)
- [ ] Power tuning: kernel 6.17+ / C10 / ASPM (research/15-power-efficiency.md)
- [ ] ZFS migration on fresh 1TB (research/19-zfs-deep-dive.md)
- [ ] Monitoring stack: pve-exporter + Prometheus + Grafana + ntfy (research/22-monitoring-stack.md)

## Security

- [x] Store darko@pve password + API token in Bitwarden (item `ProxDev - Proxmox VE`)
- [ ] Move remaining hardcoded credentials in notes into Bitwarden placeholders
- [ ] Review/scope API token privileges
- [ ] Enable TFA for darko@pve?