---
tags: [research, media, arr, jellyfin, gluetun, nas, smb, nfs]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Media (*arr) Stack + NAS/SMB Sharing

Research date: 2026-09-17.

## The stack

- Downloader: qBittorrent (linuxserver) inside gluetun's netns (`network_mode: service:gluetun`) = automatic kill switch.
- gluetun supports AirVPN, CyberGhost, ExpressVPN, IPVanish, Mullvad, NordVPN, PIA, ProtonVPN, PureVPN, Surfshark, TorGuard, Windscribe, VyprVPN + custom WireGuard. Pick one WITH port-forwarding (Proton, AirVPN, PIA, TorGuard) for good ratios; Mullvad dropped forwarding.
- Indexers: Prowlarr (+ FlareSolverr for Cloudflare-walled publics). Per-indexer HTTP proxy via tags: route geo-blocked/public indexers through gluetun's HTTP proxy (HTTPPROXY=on, port 8888) - many private trackers ban VPN IPs.
- Automation: Sonarr (TV), Radarr (movies), Bazarr (subtitles), Jellyseerr (requests front, 5055). Optional: Lidarr, Mylar3.
- Hardlinks (TRaSH): one share /data, same filesystem: /data/torrents/{tv,movies} (only downloader mounts this) + /data/media/{tv,movies} (only media server mounts this); all *arrs mount whole /data so hardlinks and atomic moves work.
- PUID/PGID: one UID/GID (e.g. 1000:1000) across every container, chown /data; add group_add video/render gids for GPU.
- Naming + profiles: use TRaSH recommended naming schemes and quality profiles (WEB-1080p, HD Bluray+WEB, UHD Bluray+WEB) with Custom Formats; negative-score BR-DISK/Extras; AV1 -10000 unless wanted. Golden rule: no x265 4K grabs.

## Compose skeleton (one bridge network)

```yaml
networks: { media: {} }

services:
  gluetun:
    image: qmcgaw/gluetun
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    ports: ["8080:8080", "6881:6881/tcp", "6881:6881/udp"]
    environment:
      - VPN_SERVICE_PROVIDER=protonvpn
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WG_KEY}
      - SERVER_COUNTRIES=Netherlands
      - FIREWALL_OUTBOUND_SUBNETS=192.168.1.0/24
      - HTTPPROXY=on
    networks: [media]

  qbittorrent:
    image: linuxserver/qbittorrent
    network_mode: "service:gluetun"
    depends_on: [gluetun]
    environment: { PUID: 1000, PGID: 1000, TZ: "Europe/Berlin" }
    volumes:
      - ./appdata/qbt:/config
      - /data/torrents:/data/torrents

  prowlarr:     # ports 9696
    image: linuxserver/prowlarr
    volumes: [./appdata/prowlarr:/config]

  sonarr:       # radarr 7878, bazarr 6767 mirror this
    image: linuxserver/sonarr
    volumes:
      - ./appdata/sonarr:/config
      - /data:/data                  # whole /data -> hardlinks
    ports: ["8989:8989"]

  jellyfin:
    image: jellyfin/jellyfin
    group_add: ["44","104"]
    devices: [/dev/dri/renderD128:/dev/dri/renderD128]
    environment: { NVIDIA_VISIBLE_DEVICES: all, NVIDIA_DRIVER_CAPABILITIES: video,transcode }
    volumes:
      - ./appdata/jellyfin:/config
      - /data/media:/data/media
    ports: ["8096:8096", "7359:7359/udp", "1900:1900/udp"]

  jellyseerr:   # 5055
    image: fallenbagel/jellyseerr
    volumes: [./appdata/jellyseerr:/app/config]
```

## Hardware (Arrow Lake + RTX 5060)

- Transcode with the RTX 5060 NVENC (Blackwell = current best H.264/HEVC/AV1 encoder), NOT the CPU. Needs nvidia-container-toolkit on host for the VM; GPU via LXC device passthrough or VM passthrough.
- Arrow Lake 200S non-F SKUs ship 4-Xe-core Xe2 iGPU (QuickSync/VA-API) but need kernel 6.12+ xe driver; boards/BIOS may disable. 275HX is HX (mobile) - check.
- In-home reality: on gigabit LAN, Jellyfin clients direct-play virtually everything (REMUX is nothing). Transcoding only matters remote/mobile or non-passthrough audio. Don't re-encode your library.

## Storage layout (1TB NVMe + 256GB)

- 256GB = Proxmox OS + VM/LXC root images + /docker/appdata (what you back up).
- 1TB NVMe = dedicated /data mount (ext4 or ZFS dataset, recordsize 1M) delivered to the media LXC via bind/fstab. Never co-locate bulk media with VM images.
- External USB/NAS later: NFS to Linux clients/Docker (~25-30% faster random reads); SMB3 for Windows/macOS. Run both. Permissions: one user, one media group, sgid dirs, chmod 2775, SMB force group = media, create mask 0664.
- NFO: enable Bazarr + Radarr/Sonarr NFO sidecars so libraries are rebuildable.

## Automation / backup

- Tdarr/unmanic optional; don't re-encode by default (quality loss > space saved at home). 
- Never back up /data/media (re-grabbable). Use PBS on the LXC/VM: nightly, dedupe, appdata + NFO only. Media to external repo later via rsync if you want a safety net.

## Security

- Publish nothing to the internet; WireGuard/Tailscale into the LAN, don't port-forward.
- qBittorrent + VPN-requiring indexers through gluetun only; private-tracker indexers on home IP.
- Jellyseerr is the public-facing front; *arr admin UIs stay LAN-only or behind reverse proxy + Authelia forward-auth. Run the whole stack as unprivileged Proxmox LXC (Docker-in-LXC).

## Sources

- https://trash-guides.info/File-and-Folder-Structure/Hardlinks-and-Instant-Moves
- https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/
- https://trash-guides.info/Radarr/radarr-setup-quality-profiles/
- https://hub.docker.com/r/qmcgaw/gluetun
- https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/nvidia/
- https://jellyfin.org/docs/general/administration/storage