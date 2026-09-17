---
tags: [research, homelab, services, selfhosted, catalog]
created: 2026-09-17 21:34:13 +05
modified: 2026-09-17 21:42:45 +05
---

# Popular Homelab Services Catalog

Research date: 2026-09-17.

Deploy legend: **LXC** = native unprivileged LXC | **LXC+Docker** = Docker in LXC (nesting=1) | **VM+Docker** = Debian/Ubuntu VM running Docker | **HAOS VM** = Home Assistant OS VM.

## Service catalog

| Service | Category | Bottom line | Deploy as | Footprint (RAM) |
|---------|----------|-------------|-----------|------------------|
| AdGuard Home | DNS | Best all-in-one DNS ad-block + DHCP; the 2025-26 default | LXC (native) | 50-150MB |
| Pi-hole | DNS | Classic network ad-blocker; pair with Unbound | LXC (native) | 80-200MB |
| Unbound | DNS | Recursive resolver backend for Pi-hole/AGH | LXC (native) | 30-60MB |
| Technitium | DNS | Rising favorite; native clustering since v14 | LXC (native) | 50-150MB |
| nginx proxy manager | Proxy | Easiest GUI reverse proxy + Let's Encrypt | LXC+Docker | 200-400MB |
| Caddy | Proxy | Simplest config, auto-HTTPS, tiny | LXC (native) | 30-80MB |
| Traefik | Proxy | IaC-style via Docker labels | VM+Docker | 100-200MB |
| Jellyfin | Media | Open, free, lightest major media server; pass /dev/dri for HW transcode | LXC (native/Docker) | 0.5-1.5GB idle |
| Plex | Media | Polished clients; heavier | VM+Docker | 1-2GB |
| Sonarr/Radarr | Media | TV/movie library automation | LXC+Docker | 200-500MB each |
| Prowlarr | Media | Central indexer manager for all *arrs | LXC+Docker | 200-400MB |
| Bazarr | Media | Auto subtitle downloads | LXC+Docker | 150-300MB |
| qBittorrent | Media | Torrent client; pair with gluetun VPN | LXC+Docker | 100-300MB |
| Jellyseerr | Media | Request UI for Jellyfin + *arrs | LXC+Docker | 200-400MB |
| Uptime Kuma | Monitor | Uptime/status page + push monitors | LXC | 150-300MB |
| Grafana + Prometheus | Monitor | Full metrics stack; add pve-exporter | LXC/Docker | 400MB-1GB total |
| Netdata | Monitor | Zero-config real-time dashboards | LXC | 200-400MB |
| Healthchecks | Monitor | Cron/heartbeat pings | LXC | 100-200MB |
| Beszel | Monitor | Lean lightweight overview (new favorite) | LXC | 50-150MB |
| Home Assistant | Home | Best as HAOS VM (Supervised + add-ons) | HAOS VM | 0.5-1.5GB |
| Node-RED | Home | Visual flow automation, pairs with HA/MQTT | LXC+Docker | 200-300MB |
| Nextcloud | Files | Full Drive substitute; PHP+DB = heavy | VM/LXC+Docker | 800MB-2GB+ |
| Syncthing | Files | Secure P2P continuous sync | LXC | 100-200MB |
| Immich | Photos | Google-Photos clone; ML spikes RAM | LXC+Docker | 0.5-1GB idle |
| Restic | Backup | Encrypted incremental snapshots | Host/VM agent | negligible |
| Vaultwarden | Secrets | Bitwarden-compatible, one tiny binary | LXC | 20-100MB idle |
| Gitea / Forgejo | Git | Lightweight self-hosted forge; Actions built in | LXC | 100-300MB idle |
| Woodpecker CI | Git/CI | Drone-style CI wired to Gitea/Forgejo | LXC+Docker | 200-500MB |
| Docker | Platform | Container runtime; official rec: VM, LXC OK with nesting=1 | VM (rec) | negligible |
| Portainer | Platform | GUI for compose stacks | LXC+Docker | 100-200MB |
| Authentik | Auth | IdP with GUI flows, self-contained | VM+Docker | 1-2GB |
| Authelia | Auth | Light SSO/MFA behind reverse proxy | LXC+Docker | 30-100MB |
| PostgreSQL | DB | Default DB for almost every stack | LXC | 100-300MB idle |
| MariaDB | DB | Lighter LAMP alternative | LXC | 150-300MB |
| Redis | DB | Cache/queue for Immich, Authentik, etc. | LXC | 50-100MB |
| SQLite | DB | Embedded; fine for Vaultwarden/Kuma/Gitea | inline | 0 |
| Paperless-ngx | Docs | OCR/archive; OCR spikes RAM | LXC+Docker | 300-600MB idle, 2-6GB OCR |
| Ollama | AI | Local LLM; RAM ~2x model size | LXC/VM (GPU passthrough) | 7B ~8GB, 13B ~16GB |
| Open WebUI | AI | ChatGPT-like UI for Ollama | VM+Docker | 0.5-1.5GB |
| Homepage/Dashy | Dashboard | Service dashboard with live widgets | LXC+Docker | 150-400MB |
| ntfy / Gotify | Notify | Self-hosted push notifications | LXC | 50-150MB |
| PBS | Backup | Proxmox Backup Server (dedup) for whole guests | LXC/standalone | 1-2GB |

## Proxmox-specific takeaways

- LXC is the default for most single apps (AdGuard, Vaultwarden, Gitea, DBs, Kuma): near-native, ~10-30MB overhead, first-class PBS snapshots. community-scripts.org gives one-line installs for ~600 services.
- Docker goes in a VM for multi-container stacks (media stack, Nextcloud, Immich, Gitea+CI) or anything untrusted; Docker-in-LXC (nesting=1) saves RAM but can hit kernel/permission quirks.
- PVE 9.1+ runs OCI images natively in LXC (tech preview, no Compose).
- Media/GUI: pass GPU via /dev/dri into LXC (Intel/AMD) or VM passthrough (NVIDIA); Ollama needs GPU for speed.
- Bind-mount NAS/media storage into LXCs for near-native I/O; back everything to PBS.

## Starter stack (62GB RAM / 24 cores, greenfield)

Foundation first (~16-18GB used, huge headroom):

1. AdGuard Home - LXC (1c/512MB) - DNS filtering + DHCP
2. Caddy (or NPM) - LXC+Docker (1c/1GB) - reverse proxy + TLS
3. Docker VM + Portainer - VM (6c/8GB) - container platform for the rest
4. Vaultwarden - LXC (1c/512MB) - passwords
5. Uptime Kuma - LXC (1c/512MB) - uptime/status
6. Jellyfin + Sonarr + Radarr + Prowlarr + Bazarr + qBittorrent - Docker VM (+4c/6GB) - media
7. Gitea/Forgejo - LXC (1c/1GB) - git forge
8. Home Assistant OS (2c/4GB) if smart home, else Paperless-ngx (2c/2GB)
9. PBS on spare disk for whole-guest backups from day one

## Sources

- https://community-scripts.org
- https://www.virtualizationhowto.com/2025/12/9-home-lab-services-i-would-deploy-first-on-a-fresh-proxmox-install
- https://www.virtualizationhowto.com/2025/11/complete-guide-to-proxmox-containers-in-2025-docker-vms-lxc-and-new-oci-support
- https://proxmoxr.com/blog/proxmox-docker-vs-lxc
- https://gist.github.com/gigamaster/03a65f453832987a837511e4db25b388