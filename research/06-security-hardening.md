---
tags: [research, homelab, proxmox, security, hardening]
created: 2026-09-17 21:24:01 +05
modified: 2026-09-18 18:25:03 +05
---

# Proxmox Security Hardening

Research date: 2026-09-17. Current state on OmaLaptop: darko@pve admin, root@pam disabled, root SSH blocked.

## Baseline checklist

| # | Item | Command / config |
|---|------|------------------|
| 1 | Strong unique passwords | strong ones; store in Bitwarden |
| 2 | TFA (TOTP) for admin | `pveum user tfa add darko@pve totp`; Datacenter > Options > Two Factor Policy > Required. Save recovery keys |
| 3 | API tokens minimal privs | `pveum user token add darko@pve ci --privsep 1 --expire <ts>` then `pveum aclmod / -token 'darko@pve!ci' -role PVEVMUser` |
| 4 | SSH keys only + allowlist | see below |
| 5 | Firewall enabled | see below |

SSH hardening (/etc/ssh/sshd_config):

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AddressFamily inet
Match Address 192.0.2.0/24,10.0.0.0/24
    AllowUsers darko
```

Note: darko@pve is a pve-realm user - cannot authenticate to PAM/SSH. For SSH create a matching PAM user with sudo, or use `pveum useradd admin@pve` + ACL instead for web-only.

## PVE specifics

- pveproxy host ACLs (/etc/default/pveproxy): `ALLOW_FROM="10.0.0.0/24"`, `DENY_FROM="all"`, `POLICY="allow"`; optionally `LISTEN_IP`.
- TLS: Datacenter > ACME (Let's Encrypt) or `pvenode cert set`; or Tailscale Serve for a valid cert.
- fail2ban (PVE 8+ no rsyslog by default - use `backend = systemd`), filter proxmox `pvedaemon.*authentication failure`, jail port https/8006, maxretry 3, findtime 2d.

## Firewall pattern

/etc/pve/firewall/cluster.fw:

```
[OPTIONS]
    enable: 1
    policy_in: DROP
    policy_out: DROP
[IPSET management]
    192.0.2.0/24
[RULES]
    IN ACCEPT from management (tcp port 8006,22)
```

Keep ssh open first! Per-node: /etc/pve/nodes/prodev/host.fw; per-VM/CT: `net0: ...,firewall=1`. Defense-in-depth: PVE FW for host + OPNsense/router VM for edge.

## Network exposure

Never expose 8006/22 to the internet. Management only via VPN: Tailscale on host (`tailscale up`), `tailscale serve --bg https+insecure://localhost:8006`, combined with pveproxy ALLOW_FROM so API won't answer off-tailnet.

## Updates

- Disable pve-enterprise (no subscription); add no-subscription for trixie in /etc/apt/sources.list.d/pve-no-subscription.sources. Keep security.debian.org.
- `apt update && apt full-upgrade` monthly (or unattended-upgrades for -security); `pveupgrade` before major jumps.
- Check Proxmox bug reports before dist-upgrade (kernel regressions with GPU passthrough).

## Secrets and disk encryption

- Bitwarden for all passwords, token secrets, TFA recovery keys, LUKS passphrase.
- Rotate API tokens quarterly (create new, `pveum user token remove`).
- LUKS root not offered by installer - manual cryptsetup (aes-xts-plain64, 512-bit) + initramfs-dropbear for SSH unlock; back up LUKS header. ZFS native encryption lighter but blocks migration/replication.

## Guest hardening

- Unprivileged LXC (root->uid 100000) for anything internet-facing; privileged only offline.
- Use VMs for untrusted workloads (Docker). Per-guest firewall + QEMU guest agent.

## Common misconfigurations to avoid

| Default risk | Fix |
|--------------|-----|
| Firewall disabled | cluster.fw enable:1, drop policies |
| pveproxy binds all IPs | ALLOW_FROM/DENY_FROM + VPN |
| root SSH login / passwords | PermitRootLogin no, keys only, Match allowlist |
| No TFA | TOTP + Two Factor Policy Required |
| Self-signed cert | ACME / Tailscale Serve |
| Enterprise repo w/o subscription | no-subscription (trixie) |
| Admin role on every user | least privilege |
| Privileged LXC on internet | unprivileged + VM for untrusted |

## Sources

- https://pve.proxmox.com/pve-docs/chapter-pve-firewall.html
- https://pve.proxmox.com/pve-docs/chapter-pveum.html
- https://pve.proxmox.com/wiki/Fail2ban
- https://pve.proxmox.com/wiki/Package_Repositories
- https://tailscale.com/kb/1133/proxmox