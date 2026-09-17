---
tags: [research, networking, router, opnsense, firewall, vlan]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Virtual Router (OPNsense) on Proxmox

Research date: 2026-09-17.

## Virtual router in a homelab

Consensus: virtualizing OPNsense is homelab-positive when the firewall guards the VM estate on a single box (snapshots, config restore in minutes, DHCP/DNS/VPN on one host). It is wrong when the hypervisor ITSELF is the network's only perimeter.

[CAUTION] The classic pitfall: internet dies with the host, and if PVE management routes through the VM, a bad firewall config locks you out of the hypervisor. Mitigations: keep PVE management on its own bridge path independent of OPNsense, keep NoVNC/IPMI console access, set OPNsense startup order first with delays, PBS/config backup, UPS.

## OPNsense vs pfSense vs OpenWrt (2026)

- OPNsense (Deciso, BSD-2): community default for new x86 builds - fully open, ~fortnightly patches + 2 majors/year, native WireGuard (kernel), Suricata IPS + Unbound/AdGuard, REST API + 2FA.
- pfSense: CE 2.8 lost offline ISO (needs internet for installer); Plus is paid on non-Netgate hardware; slow updates. Only if Netgate boxes or pfBlockerNG/Snort dependency.
- OpenWrt: best for low-power/reused hardware, SQM, lean routing; weak at Suricata-scale firewall duty.
- Zenarmor: free tier = single fixed policy + reporting; per-interface/TLS inspection needs ~$10/mo Home tier. Suricata free but CPU-hungry (inline IPS forces 4+ vCPUs).

## NIC pairing (this box: fixed RTL8125 + add-on I226)

- FreeBSD's Realtek drivers are weak: `re` ~doesn't do RTL8125 well (pause bugs, link-recovery regressions, >1G issues); new `rge` immature.
- Intel I226 uses `igc` - solid, but known link-flap bug (EEE/ASPM/firmware; fix via dev.igc.N.fc=0, offloads off).
- RECOMMENDED: add a 2.5G Intel I226 NIC as the physical WAN port passed through to OPNsense (FreeBSD's strength); keep RTL8125 on the Proxmox/Linux side (Linux RTL8125 driver is excellent) for LAN + host management.

## Recommended topology (single 10.10.10.0/24 box)

- vmbr0: RTL8125, VLAN-aware (`bridge-vlan-aware yes`, `bridge-vids 2-4094`), host + LAN VM trunk. WAN traffic not here.
- I226: passed through as raw PCI device -> OPNsense WAN (physically isolated from host network stack - the one case passthrough earns its keep).
- OPNsense VM: Net0 = virtio on vmbr0 trunk (all VLANs), Net1 = passed-through I226 (WAN). VLAN subinterfaces inside OPNsense do all routing/DHCP (router-on-a-stick) - 10.10.10.x ends on a dedicated VLAN; inter-VLAN firewalled.

## Proxmox side

- q35 + BIOS/OVMF, CPU type host (AES-NI for WireGuard), 4GB RAM (8GB if Suricata/Zenarmor; no ballooning with guest agent), VirtIO-SCSI disk >=32GB, qemu-guest-agent.
- Virtio > e1000 (e1000 caps at 1Gbps; virtio does ~2.5-4.7Gbps through the VM).
- Disable PVE firewall on OPNsense vNICs; keep OPNsense offloads checked (disable hw checksum/TSO/LRO) - #1 troubleshooting answer.

## Setup (terse)

Install the `-serial.iso`; assign WAN/LAN by MAC (spoof from Proxmox side for stability), WAN=igc (static/gateway per ISP), LAN=trunk on vtnet0, create VLANs, WAN default block + NAT outbound if single static, DHCP on LAN/VLANs with DNS at AdGuard (LXC or plugin), WireGuard server on UDP 51820 + NAT port-forward.

## Migration (current PVE @ 10.10.10.152/24, gw .1)

- Keep PVE management on vmbr0/RTL8125 with 10.10.10.152 and gw 10.10.10.1 UNCHANGED during the pilot - host stays reachable regardless of OPNsense.
- Build OPNsense behind the existing router first (WAN = DHCP from .1) to validate routing/VLANs/WG; then cut the WAN NIC to the modem/ISP.
- Move guests behind OPNsense: change each guest's gateway from .1 to OPNsense LAN IP (never keep .1 - clients bypass the firewall).
- Lockout caveats: only change bridges/VLANs from the NoVNC console; do not put 10.10.10.152 behind OPNsense until a console-rescue path is proven; re-pair PVE's gateway last. Restore = import config XML or snapshot rollback.

## Sources

- https://forum.opnsense.org/index.php?topic=44159.0 (official virt HOWTO - meyergru)
- https://prohomelab.com/en/posts/router-virtualization-pros-and-cons/
- https://homelabaddiction.com/pfsense-vs-opnsense-vs-openwrt-which-router-firewall-should-you-run-in-your-homelab-in-2026/
- https://serverside.com/blog/opnsense-on-proxmox
- https://cwwk.com/blogs/firewall-networking/opnsense-proxmox-bridge-vs-pci-passthrough
- https://computingforgeeks.com/fix-intel-i226-nic-drops-opnsense/