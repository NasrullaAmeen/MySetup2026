---
tags: [research, wifi, networking, omalaptop, laptop, roaming, nat]
created: 2026-09-18 12:15:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 33. Proxmox host on Wi-Fi - what actually works (OmaLaptop)

Research date: 2026-09-18. OmaLaptop is a laptop (PHN16S-71): after a PVE
install half its life is "no ethernet port free" - home router port used,
or on the road (phone hotspot). plan/networking.md already picked a path
(host owns Wi-Fi, VMs behind NAT, never bridge Wi-Fi). This note digs into
WHY that is the right call, what the alternatives really are (2026 facts),
and the reliability details that make a Wi-Fi-uplinked hypervisor usable.

## TL;DR - the decision for OmaLaptop

- **Wired uplink (home dock / Killer E3000):** host on 10.10.10.x, and VMs
  MAY be bridged onto the LAN (Ethernet is a real L2 medium) - or stay NAT
  if you prefer isolation. Plan keeps NAT for now.
- **Wi-Fi uplink (away / fallback):** host joins Wi-Fi itself; VMs ALWAYS
  sit on a private vmbr/NAT net. Bridging is off the table (below).
- Extra Internet for VMs = one masquerade rule + ip_forward (proven by many
  laptop-PVE posts, 2026 included).

## Why Linux refuses to put a Wi-Fi client in a bridge

802.11 station frames have **3 MAC addresses** (radio source/dest + final
AP/Host) and assume exactly one originator. An Ethernet bridge must learn
and forward frames by arbitrary source MAC - impossible, and more
importantly the **AP filters**: it only accepts/forwards frames whose
source MAC is the one it authenticated. So a VM behind wlan in a bridge =
frames silently dropped (or `brctl addif` fails with "Operation not
supported"). This is a client-card + AP constraint, not a Proxmox bug.

### Escape hatches (in order of fuss)

| Approach | How | Needs | Verdict for OmaLaptop |
|---|---|---|---|
| **NAT (masquerade)** | vmbr0 private net + `iptables POSTROUTING -o wlan -j MASQUERADE`, `ip_forward=1` | nothing else | **Standard. Use this.** |
| Routed + router static routes | host routes VM subnet to router via AP; router gets route to 10.20.0.0/24 | router supports static routes/DHCP options | try only if NAT blocks LAN-reach; home router may not support it |
| ARP proxy (`parprouted` + `proxy_arp`) | wlan and vmbr0 both get IPs; proxy ARP glues them at L3 | unicast only, no multicast (mDNS dies), DHCP relay for guests | niche; NAT is simpler |
| WDS/4addr bridge | `iw dev wlan set 4addr on` + AP must enable WDS (`wds_sta=1` on hostapd) | driver support AND AP cooperation | Intel `iwlwifi` **no 4addr in station mode**; OmaLaptop card excluded |

Explicit 4addr note (2023+ wpa_supplicant): there is now an
`enable_4addr_mode` per-network option and you can create a second station
vif (`iw phy ... interface add type managed 4addr on`), but the AP must
still opt in, and iwlwifi station mode does not implement 4addr - so it is
a dead end for this laptop's CNVi card regardless of software.

## Recommended OmaLaptop config (Wi-Fi path)

Host owns the card. PVE ships Debian ifupdown; two viable managers:

1. **wpa_supplicant + ifupdown** (classic, scriptable, matches PVE systemd
   model). Multi-SSID:
   ```
   network={ ssid="Home 5G" psk="..." priority=10 }
   network={ ssid="iPhone"  psk="..." }   # hotspot - highest priority on top
   ```
   autoscan directive for background roaming; `ap_scan=1`.
2. **NetworkManager** (2026 laptop-PVE scripts like Davinci198 prefer NM):
   easier trunk config and auto-reconnect; but it fights ifupdown for any
   bridge - keep NM OUT of vmbr0 (mark it unmanaged, or let NM own only
   wlan). Choose ONE manager. Plan already targets wpa_supplicant; stick
   with it.

`/etc/network/interfaces` (Wi-Fi uplink case):
```
allow-hotplug wlp128s20f3
iface wlp128s20f3 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

auto vmbr0
iface vmbr0 inet static
    address 10.20.0.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up   iptables -t nat -A POSTROUTING -s 10.20.0.0/24 -o wlp128s20f3 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s 10.20.0.0/24 -o wlp128s20f3 -j MASQUERADE
```
plus `net.ipv4.ip_forward=1`. VMs: static 10.20.0.x, gw 10.20.0.1.

### DHCP + hosts hygiene

- Give the host a **DHCP reservation** on the home AP; wireless DHCP works
  like wired once Wi-Fi is up. Put that IP in `/etc/hosts` (pveproxy
  complains about unresolvable hostname otherwise) - some guides set the
  Wi-Fi IP as the hostname IP instead of the wired one.
- Wi-Fi drops = PVE web UI + host unreachable. Fix = reconnect logic, not
  more config (below).

## Reliability for a roaming hypervisor

- **Wi-Fi watchdog** (2026 pattern): a tiny systemd unit pings the gateway;
  on N fails it `ip link set wlan down/up` (or `nmcli` re-associate),
  restarting wpa_supplicant if the card is stuck. e.g. Davinci198's setup
  uses exactly this. Without it a dead association silently kills all VMs'
  Internet until reboot.
- `iw dev wlan set power_save off` - laptop Wi-Fi power-management adds
  latency and can drop packets; important because GameWin11 games flow over
  this same uplink. Latency penalty is uplink-wide, not a VM-net bug.
- **Captive portals** bite hard here: host needs network before you can
  click "agree". Planning already notes: use phone tethering / curl
  login in the hotspot window.
- `linux-firmware` updates: Intel cards (CNVi, Wi-Fi 7 BE200) silently lose
  features on vanilla PVE until you `apt install linux-firmware` (recent
  iwlwifi fw). Add to Phase A tooling. Country code `MV` already set.
- Roam between two APs (mesh): `autoscan` + min rssi tune; expect a blip -
  that is normal for Wi-Fi roaming, NAT re-establishes quickly.

## Known trade-offs (accept them)

- VM <-> LAN device talks (printer, router-NAS, ProxLab by IP) must go
  through host port-forwards or a routed config; NAT by itself is
  outbound-only. Plan's "expose only when docked" rule covers this.
- No transparent L2 for VMs on Wi-Fi: re-running tough for things like
  ARP-snooping tools inside VMs. Irrelevant here.
- mDNS/Avahi broadcast crossing to VMs: does not traverse NAT (unicast
  only). Media VMs (research/20) should target NAS by IP, not discovery.

## Reference implementations consulted (2026)

- Forum HOWTO "PVE 8 PVE 9 wifi routed" - full routing needs a capable
  router (static routes); otherwise SNAT is the default and easiest.
- findichgut.net "Using Proxmox over WiFi" - ARP-proxy (parprouted) walk-
  through and hybrid wired/wifi configs.
- Davinci198/Proxmox-VE-Laptop-Hybrid-Setup (2026-04) - NetworkManager +
  dnsmasq on vmbr1 + WiFi watchdog + `power_save off` + TLP, PVE 9.
- Terrence Miao "Proxmox over Wireless" (2025) - Wi-Fi 7 / BE200 firmware
  on PVE 9.1: vanilla PVE lacks fw; `linux-firmware` needed.
- Kernel docs (iw/mac80211): multi-vif station + 4addr support matrix;
  wpa_supplicant `enable_4addr_mode` patch (hostap list).

## Open questions

- Home uplink normally = Killer E3000 wired; do we want a "dock = bridged
  LAN for VMs, undocked = NAT" dual config, or keep NAT always (current
  plan)? Keep NAT always - one config, zero surprises. Settled.
- Watchdog: add a systemd `wifi-watchdog` unit in Phase A (alongside E4
  power tuning) - it is cheap insurance once Wi-Fi roaming starts.