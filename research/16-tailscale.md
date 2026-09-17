---
tags: [research, tailscale, vpn, remote-access, subnet-router, loopoff]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Tailscale Deep-Dive for Remote Access

Research date: 2026-09-17.

## Basics

- Identity-based overlay mesh VPN on WireGuard. Control plane coordinates NAT traversal (STUN + DERP relay fallback); nodes punch direct tunnels and get a 100.x.y.z IP.
- No inbound port-forwarding, no cert management, no public endpoint. Tailnet = your private network of nodes + virtual subnets.
- 2026 pricing: free Personal plan = 6 users, unlimited user devices, up to 50 tagged resources, 3 ACL groups. Subnet routers/exit nodes free.

## Install: host vs LXC

- Host install works but is discouraged: third-party software on the hypervisor + DNS/route conflicts risk with PVE's patched Debian base.
- Recommended: minimal unprivileged Debian LXC with TUN access. Add to /etc/pve/lxc/<id>.conf:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Then `systemctl enable now tailscaled` + `tailscale up` (browser login). Enable MagicDNS + HTTPS in the admin console; disable key expiry on servers.

## Subnet router for 10.10.10.0/24

```
# in the LXC:
sysctl -w net.ipv4.ip_forward=1
tailscale up --advertise-routes=10.10.10.0/24 --advertise-tags=tag:router
```

- Approve the route in the admin console (Machines > edit routes). Clients must enable Accept routes (`tailscale set --accept-routes`).
- Verify with `ts-netcheck` and `tailscale status` (direct vs relayed `via`).
- On some Linux clients the route lands in policy table 52 and needs `ip rule add to 10.10.10.0/24 pref 5000 lookup 52`.

## serve / funnel / ACLs / tags

- `tailscale serve --bg https+insecure://localhost:8006` exposes the Proxmox UI tailnet-wide on 443 with an auto cert (valid *.ts.net). [INFO] Prefer Serve (private); keep Funnel off unless you want public (Cloudflare Tunnel is the better public option - TLS-MITM and media-streaming ToS caveats).
- Tag infrastructure nodes (tag:server, tag:router) via --advertise-tags so ACLs target tags, not user devices.
- Exit nodes: `--advertise-exit-node` on the LXC lets devices route traffic out through the homelab.
- Provision via reusable tskey-auth- tailnet keys / ephemeral keys.

## Proxmox UI + SSH

- With the subnet router: https://10.10.10.x:8006 (still self-signed) or `tailscale serve` for a valid cert. Alternative: `tailscale cert <node>` + `pvenode cert set`.
- SSH works directly over tailnet IP / MagicDNS name.

## Recommended topology (10.10.10.0/24)

Dedicated unprivileged Debian LXC (or the AdGuard LXC once stable) at a static 10.10.10.x, tagged tag:router, advertising 10.10.10.0/24 (+ --advertise-exit-node optional). ACL policy:

```
"acls": [
  { "action":"accept", "src":["tag:router"], "dst":["*:*"] },
  { "action":"accept", "src":["darko@github.com"], "dst":["tag:proxmox:8006,22","10.10.10.0/24:8006,22"] }
]
```

Keeps the hypervisor clean, one choke point, only darko + devices reach 8006/SSH.

## Sources

- https://tailscale.com/docs/integrations/proxmox
- https://tailscale.com/docs/features/subnet-routers
- https://tailscale.com/pricing
- https://idif.net/posts/tailscale-subnet-router
- https://github.com/tailscale/tsidp