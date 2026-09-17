---
tags: [research, proxmox, community-scripts, lxc, helpers]
created: 2026-09-17 21:34:13 +05
modified: 2026-09-17 21:42:45 +05
---

# community-scripts.org (Proxmox VE Helper-Scripts)

Research date: 2026-09-17.

## Project status (2026)

- One-liner installer project created by the late tteck; since Nov 2024 maintained as the community fork `community-scripts/ProxmoxVE`.
- ~29,100 stars / 2,800 forks, MIT license. Maintained by volunteers (MickLesk, michelroegl-brunner, BramSuurdje, CrazyWolf13, tremor021, vhsdream, asylumexp).
- Supports PVE 8.4 / 9.0 / 9.1 / 9.2. Steady activity; releases/changelogs dated Aug-Sep 2026; breaking-changes advisories exist.

## What it provides

- ct/ (LXC app containers): Home Assistant, Zigbee2MQTT, ESPHome, Node-RED, Jellyfin, Plex, Radarr/Sonarr, Immich, Paperless-ngx, Nextcloud, Gitea, Vaultwarden, AdGuard/Pi-hole, Nginx Proxy Manager, Ollama, Open WebUI, Grafana, PostgreSQL, MariaDB, Redis, n8n, Portainer, and ~600 more.
- Alpine variants (ct/alpine-*.sh): being migrated to Debian CTs (breaking change Aug 2026).
- VM scripts (vm/): Home Assistant OS, Debian/Ubuntu helper VMs, openwrt/pfSense-oriented helpers.
- Sandbox flag on newer/experimental scripts.
- Host tools (tools/pve/*): cronmaster, iptag, clean-lxcs, update_lxcs, networkoptimizer, pve-scripts-local.

## Usage

Standard currently uses curl (not wget), run in the Proxmox root shell:

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/ollama.sh)"
```

- Prompts Default vs Advanced; Default = sensible resources (e.g., Ollama: 4 CPU / 4096 RAM / 40GB), Advanced = multi-step wizard (~20-28 steps).
- Overrides via exported `var_*` env vars:

```
export var_cpu=4 var_ram=4096 var_disk=40 var_hostname=myapp
export var_net=dhcp var_gateway=192.168.1.1 var_mtu=1500
export var_ns=8.8.8.8 var_mac=02:00:00:00:00:01
export var_os=debian var_version=13 var_unprivileged=1
export var_gpu=yes var_verbose=yes var_tags=prod
export var_brg var_vlan var_ssh var_pw var_nesting var_keyctl var_fuse var_tun var_timezone var_container_storage
```

## Behavior

- Creates an unprivileged LXC (nesting on) via pct create, then runs container-side install scripts (OS setup -> deps -> app -> config -> DB -> services -> version tracking -> cleanup), prints the app URL/IP at the end.
- Optional Tailscale, auto-reboot toggle, per-app update vs reinstall scripts.

## Safety

[CAUTION] You are piping a remote GitHub script to bash - download and eyeball the script and referenced core/ files before running. Official disclaimer: "use at your own risk", MIT, runs as root on the hypervisor.
- Scripts do resource/disk checks and container-ID validation; hold/pin versions for some apps (Immich, Mealie).
- Uninstall = delete the CT/VM. Uses well-known upstream sources (GitHub releases, official repos), not random tarballs.

## Popular scripts

| Script | Category | Type |
|--------|----------|------|
| Home Assistant | Home Automation | LXC |
| Vaultwarden | Security/Password | LXC |
| AdGuard Home | Networking/DNS | LXC |
| Pi-hole + Unbound | Networking/DNS | LXC |
| Nginx Proxy Manager | Proxy | LXC |
| Jellyfin | Media | LXC (GPU) |
| Immich | Photo/Media | LXC |
| Paperless-ngx | Docs | LXC |
| Nextcloud | Cloud/Files | LXC |
| Gitea | Dev | LXC |
| Ollama | AI/LLM | LXC (GPU) |
| Open WebUI | AI/LLM | LXC |
| Grafana + Prometheus | Monitoring | LXC |
| PostgreSQL / MariaDB | Databases | LXC |
| n8n | Automation | LXC |

## Recent additions (2026)

Nagios, Neko, StoryBook, Sportarr, Kutt, Qui, Fladder, Jellystat, GWN-Manager, Whisparr, DroppedNeedle. LLM/GPU: Ollama (v0.34.x), Open WebUI, AnythingLLM, Unsloth, Lemonade Server; NVIDIA GPU helpers improved for 5000-series.

## Sources

- https://community-scripts.org
- https://github.com/community-scripts/ProxmoxVE
- https://community-scripts.org/docs (config reference, advanced settings, technical reference)
- https://community-scripts.org/breaking-changes
- https://github.com/community-scripts/ProxmoxVE/releases