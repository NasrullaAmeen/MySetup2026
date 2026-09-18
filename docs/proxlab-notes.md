---
tags: [proxmox, credentials, server, access, admin, proxlab]
created: 2026-09-18 10:30:00 +05
modified: 2026-09-18 18:25:03 +05
---

# ProxLab Server Notes

## Status

| Item | Value |
|------|-------|
| Role | Stationary homelab Proxmox host |
| Current | Omarchy (Arch) workstation, hostname `omarchy` - **PVE NOT YET INSTALLED** |
| Planned MGMT | https://10.10.30.1:8006 on 10.10.30.0/24 |
| Internal VM bridge | vmbr0 10.30.0.0/24 - gw 10.30.0.1 |
| Router | 10.10.10.1 routes 10.10.10.0/24 <-> 10.10.30.0/24 |
| Hardware | see [proxlab-hardware.md](./proxlab-hardware.md) |

- This is the box I captured hardware from on 2026-09-18. It still runs the
  daily-driver Omarchy desktop; converting it to ProxLab is a planned task
  (docs/tasks.md), not done yet.

## Roadmap to ProxLab (from "This PC will become ProxLab")

1. [ ] Back up / migrate any local data off nvme0n1 (Omarchy root) that must survive.
2. [ ] Install Proxmox VE on nvme0n1 (Lexar 500G), like OmaLaptop: `pve-no-subscription`
       repo, `darko@pve` administrator instead of root@pam, root SSH disabled.
3. [ ] Set MGMT ip 10.10.30.1/24 on 10.10.30.0/24 (router 10.10.10.1).
4. [ ] Build storage per research/26: LVM-thin on WD SSD (VM pool), ZFS mirror
       of the 2 Seagate HDDs (NAS), PNY USB stays detached backup.
5. [ ] Desktop VM with RX 5700 XT passthrough (vendor-reset, research/25).
6. [ ] Adopt cross-host backup scheme (research/28) + decide cluster (research/27).
7. [ ] Rename hostname `omarchy` -> `proxlab`.

## Access (placeholder until PVE install)

> No credentials yet - none exist for a ProxLab PVE host. When created insist on:
> - user `darko@pve` (Administrator, ACL on /) - **not** `root@pam` web login
> - API token `darko@pve!clitoken` for scripts (Bitwarden item)
> - root SSH: `PermitRootLogin no`, key-only, like OmaLaptop (research/06).

## Network / security targets (mirror OmaLaptop)

- [ ] TFA (TOTP) for darko@pve (research/21-tfa-sso.md)
- [ ] Reverse proxy + TLS (Caddy) for exposed services (research/17)
- [ ] Firewall: per-host rules; only explicit cross-traffic ProxLab <-> OmaLaptop
      (plan/proxlab.md)

## TODO

- [x] Hardware stack captured -> [proxlab-hardware.md](./proxlab-hardware.md)
- [ ] Save any Omarchy data before the PVE wipe
- [ ] Install PVE on nvme0n1 (follow plan/omalaptop.md runbook + this roadmap)