---
tags: [proxmox, credentials, omarchy, omalaptop, laptop, access]
created: 2026-09-17 20:28:39 +05
modified: 2026-09-18 18:25:03 +05
---

# OmaLaptop - Omarchy laptop notes

**OmaLaptop** is the PHN16S-71 laptop (Acer Predator). It was formerly
"ProxDev" (renamed 2026-09-18). After the pivot it is an Omarchy desktop host
on the iGPU with virt-manager outer VMs and a nested ProxDev PVE VM
for the lab tier. Authoritative blueprint: plan/omalaptop.md; nested PVE
research: research/34, 36; Omarchy/agent/mise: research/40.

## STATUS (2026-09-18)

- **PIVOTED**: bare-metal PVE retired. The laptop will be wiped to Omarchy
  on nvme0n1 (WD SN520 256G, LUKS + btrfs + Limine). World-reachable
  addresses were transferred to the Omarchy-era below.
- Task A1 is the pending manual step: wipe old PVE, install Omarchy.

## Machine

| Item | Value |
|---|---|
| Host | OmaLaptop (was ProxDev) |
| PVE era node id | `omalaptop` (retired bare-metal Proxmox VE 9.2.20) |
| Omarchy era | 2026-09-18 -> Omarchy desktop (iGPU) + virt-manager + nested PVE VM |

## Network (2026-09-18)

- Home mgmt IP **10.10.10.10** now lives on OmaLaptop (reuses old PVE mgmt IP).
  The laptop is DHCP on Wi-Fi, so 10.10.10.10 must be DHCP-reserved on the
  router (10.10.10.1) by MAC, or set statically.
- Host DNAT: `10.10.10.10:8006 -> 10.20.0.30:8006` keeps the old mgmt URL
  pointing at the nested ProxDev PVE VM. (tasks.md A8 / A9)
- Inner bridge 10.20.0.0/24 (NAT, host-owned; Wi-Fi never bridged):

| Device | Address | Purpose |
|---|---|---|
| Host vmbr0 (OmaLaptop) | 10.20.0.1/24 | inner gateway + NAT |
| MainArch (MainVM) | 10.20.0.10/24 | main desktop/manage VM |
| GameWin11 | 10.20.0.11/24 | dGPU (RTX 5060 VFIO) |
| MainWin11 | 10.20.0.12/24 | iGPU-swap MainOS |
| ProxDev (nested PVE VM) | 10.20.0.30/24 | :8006 via host DNAT |
| DevWin/DevArch/DevDeb/DevMac | 10.20.0.20-23/24 | dev tier |
| DNS | 1.1.1.1, 8.8.8.8 | upstream |

- Uplink address follows each network (10.10.10.x at home, 192.168.43.x on a
  hotspot); the inner range never changes. Details: plan/networking.md.

## Host stack (OmaLaptop = Omarchy + virt)

- Omarchy on nvme0n1; kernel cmdline in boot/limine.conf (not GRUB); updates
  via `omarchy update`. Hugely relevant: research/40 (Omarchy / agent / mise).
- Virt: `qemu-desktop libvirt virt-manager dnsmasq edk2-ovmf swtpm
  iptables-nft tpm2-tools`; enable `libvirtd virtlogd`.
- GPU rules (hard): iGPU host-only (research/37); dGPU VFIO only at the outer
  level; NO `iommu=pt`; `vfio-pci.disable_idle_d3=1` + udev D0; one GPU VM
  start per host boot (Blackwell reset, research/38). No GPU inside nested PVE.
- Stored VMs: hynix 1TB (hynix P41) = host KVM pool (thin qcow2). No virtual
  disks on the WD SN520 boot drive.

## Hardware

- See [omalaptop-hardware.md](./omalaptop-hardware.md) - full PHN16S-71 stack.
  Highlights: Core Ultra 9 275HX (24C/24T), 62.2 GB RAM, RTX 5060 Max-Q
  (VFIO), iGPU drives the desktop.

## Retired bare-metal PVE era (kept for record)

- **Proxmox VE 9.2.20** (release 9.2), web UI https://10.10.10.10:8006/, node
  `omalaptop`. Wiped by A1; entries below document how the box was administrated
  while still bare-metal.

### Access / accounts (retired era)

> Credentials stored in Bitwarden - vault item: `OmaLaptop - Proxmox VE` (id
> `2cc66d2e-18e9-4aec-8c8c-b4c801087393`). Usernames/tokens below are redacted;
> get real values from Bitwarden (`bw get item 2cc66d2e-...`).

| User | Realm | Role | Status |
|---|---|---|---|
| darko@pve | pve | Administrator (ACL / propagate) | enabled (retired with box) |
| root@pam | pam | Administrator | DISABLED (2026-09-17) |

- Password + API token (`darko@pve!clitoken`) in Bitwarden.
- root SSH disabled 2026-09-17 (`PermitRootLogin no`, reloaded, verified;
  backup `/etc/ssh/sshd_config.bak.20260917`).

### API usage pattern (apply to the nested PVE later)

```bash
curl -sk -H "Authorization: PVEAPIToken=<darko@pve!clitoken>=<secret>" \
  https://10.20.0.30:8006/api2/json/version
```

## Omarchy agent on OmaLaptop

- After A1 the laptop gets the same agentic desktop as research/40 documents:
  mise stubs in `~/.local/bin/` (claude, codex, opencode, gemini, copilot,
  etc.), `omarchy default agent`, `~/.agents/skills` (omarchy + diagnose-crash),
  `omarchy update` keeps stubs + toolchains current. This exact repo is
  worked on from such a stack.

## TODO

- [ ] A1: wipe PVE -> Omarchy on nvme0n1 (manual; user).
- [ ] A8/A9: DHCP-reserve 10.10.10.10 on router; host DNAT :8006 -> 10.20.0.30.
- [ ] Recreate darko@pve + API token inside the nested PVE (fresh install).