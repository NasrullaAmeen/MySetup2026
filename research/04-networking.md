---
tags: [research, homelab, proxmox, networking, vlan, firewall, tailscale]
created: 2026-09-17 21:24:01 +05
modified: 2026-09-17 21:42:45 +05
---

# Homelab Networking - VLANs, Firewall, Tailscale

Research date: 2026-09-17. Target: home network 10.10.10.0/24, prodev 10.10.10.10, 2.5GbE Realtek + onboard NICs.

## Proxmox networking model

- `vmbrX` bridge = soft Layer-2 switch; VMs/LXCs plug via tap/veth; NIC/bond is the uplink. Linux bridge default (recommended); avoid OVS (tap-port drops on reboot).
- Bonding: 802.3ad only if switch supports LACP, else active-backup; pointless on a single 2.5GbE port.
- VLANs: prefer a single VLAN-aware bridge (`bridge-vlan-aware yes` + `bridge-vids`), tag per VM in GUI. Trunk port on switch must permit all VLANs.

## Recommended VLAN layout

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 1 | LAN/Native | 10.10.10.0/24 | trusted devices (PVID) |
| 10 | MGMT | 10.10.20.0/24 | PVE hosts, ssh/8006 |
| 20 | SRV | 10.10.30.0/24 | VM workloads |
| 30 | IoT | 10.10.40.0/24 | smart devices isolated |
| 40 | DMZ/GUEST | 10.10.50.0/24 | internet-facing |

Config example (/etc/network/interfaces):

```
auto enp2s0
iface enp2s0 inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 20 30 40

auto vmbr0.10
iface vmbr0.10 inet static
    address 10.10.20.10/24
    gateway 10.10.20.1
```

## Firewall

Layered: PVE built-in firewall (nftables) for per-host/per-VM, default DROP, protects host independent of guest OS; a router VM (OPNsense/pfSense) does inter-VLAN routing/filtering. Run *sense as a VM only if host reboot during network maintenance is acceptable (some prefer a dedicated mini-PC). PVE host mgmt (8006/22) sits outside the router VM - lock it with PVE firewall + mgmt rules.

## Tailscale

Recommended: install on the PVE host, advertise subnet routes:

```
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale set --advertise-routes=10.10.10.0/24,10.10.20.0/24
```

Then reach VMs/API from anywhere without port-forwarding. WebUI with valid cert: `tailscale serve --bg https+insecure://localhost:8006`. Alternative: Tailscale per VM/LXC. Caveat: on a flat VLAN, one compromised Tailscale VM = lateral movement - isolate first (VLANs), then expose.

## 2.5GbE notes

- Use predictable names (eno1/enp2s0); kernel updates can rename interfaces - PVE 9 offers `proxmox-network-interface-pinning`.
- Realtek RTL8125: kernel r8169 works, but r8125 driver is common for reliability; blacklist one.
- Jumbo frames (MTU 9000): skip for a homelab on 2.5GbE - marginal gains, MTU mismatch is a classic silent breakage. Only for iSCSI/NFS storage networks.
- WiFi never bridges into vmbr for VMs (802.11 4-addr mode breaks) - use NAT/masquerade.

## DNS

Run AdGuard Home (or Pi-hole) in its own LXC; static IP via DHCP reservation. PVE host should not run systemd-resolved (conflicts on port 53). AdGuard Home has built-in DoH/DoT; Pi-hole needs unbound (cloudflared retired 2026).

## Pitfalls

- After network edits keep OOB console open; apply to `interfaces.new` then `ifreload -a`.
- VLAN tag mismatch: link up but no DHCP - looks like dead bridge.
- Two VMs with same static IP fight; use DHCP reservations as single source of truth.
- Do not set IP on `bridge-ports none` bridge expecting LAN reachability.
- Keep cluster/storage traffic off noisy flat VLAN if clustering later.

## Sources

- https://pve.proxmox.com/wiki/Network_Configuration
- https://pve.proxmox.com/wiki/Firewall
- https://tailscale.com/kb/1133/proxmox
- https://forum.proxmox.com/threads/networking-best-practice.163550
- https://github.com/kuz-lab/homelab