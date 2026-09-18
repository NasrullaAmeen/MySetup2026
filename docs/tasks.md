---
tags: [tasks, todo, docs]
created: 2026-09-17 20:53:22 +05
modified: 2026-09-18 18:25:03 +05
---

# Tasks

Ordered OmaLaptop roadmap (PIVOTED 2026-09-18: Omarchy desktop host + nested
PVE lab VM, see plan/omalaptop.md). Each phase top-to-bottom; move on only when
the current task is verified.

## OmaLaptop - Phase A: Omarchy host (do first)

- [ ] A1 Install Omarchy on nvme0n1 (WD 256G), iGPU desktop confirmed.
- [ ] A2 Virt stack: `qemu-desktop libvirt virt-manager dnsmasq edk2-ovmf
      swtpm iptables-nft tpm2-tools`; enable libvirtd + virtlogd; default
      net replaced by host NAT bridge 10.20.0.0/24 (networking.md).
- [ ] A3 IOMMU verified: kernel cmdline `intel_iommu=on` (NO `iommu=pt` -
      Blackwell, research/38), `vfio-pci.disable_idle_d3=1` + udev D0 rule;
      dGPU (10de:2d59 + 10de:22eb, group 14) via vfio-pci.ids; dmesg VT-d.
- [ ] A4 Nested enabled: `options kvm_intel nested=1` + module reload;
      CPU host-passthrough test guest boots (research/34).
- [ ] A5 Storage: nvme0n1 = OS/ISO; nvme1n1 (hynix) = KVM pool dir for thin
      qcow2; baseline tooling, timezone, git clone of this repo (research/30
      wearout in mind).
- [ ] A6 Wi-Fi: linux-firmware, multi-SSID wpa_supplicant (MV), `power_save
      off`, wifi-watchdog (networking.md sec 18; research/33).
- [ ] A7 First outer throwaway VM on the NAT bridge - desktop + VM have
      Internet while host is on Wi-Fi (networking.md runbook).
- [ ] A8 Home mgmt IP: DHCP-reserve 10.10.10.10 for the host on router
      10.10.10.1 (host is DHCP on Wi-Fi); prep host DNAT
      `10.10.10.10:8006 -> 10.20.0.30:8006` for later nested PVE access
      (plan/omalaptop.md 4b).

## OmaLaptop - Phase B: nested ProxDev PVE (lab tier)

- [ ] B1 Create nested guest: q35 + OVMF + host-passthrough, 8c/16G, ~80G
      thin qcow2 on hynix, virtio NIC 10.20.0.30; PVE 9.2 ISO boots
      (research/34).
- [ ] B2 PVE baseline inside guest: repos reviewed (lab rules), `apt update`,
      darko@pve + TFA is optional here; no enterprise repo needed.
- [ ] B3 PVE-internal vmbr 10.40.0.0/24 + NAT -> 10.20.0.1; lab egress +
      DNS verified from a test LXC.
- [ ] B4 AdGuard LXC (10.40.0.2) first service (research/09+10+11).
- [ ] B5 PBS available: PBS-as-LXC (10.40.0.3) OR vzdump to 10.20.0.x/PNY;
      config backup + first restore drill (research/05, 32).

## OmaLaptop - Phase C: outer GPU VMs (virt-manager)

- [ ] C1 GameWin11: q35/OVMF/vTPM, RTX 5060 group 14 (VFIO) + audio, Win11,
      NVIDIA driver + reset quirk (research/02, 29, 38). Start manual; ONE
      GPU VM run per host boot (Blackwell reset, research/38).
- [ ] C2 MainWin11: same minus VFIO (virtio-gpu); document the
      GameWin11 <-> MainWin11 dGPU swap (research/29).
- [ ] C3 Session resilience: VM + nested PVE survive host S3 resume (nested
      PVE reboot recipe); USB passthrough for controller/VR etc.

## OmaLaptop - Phase D: reliability + polish

- [ ] D1 Backups: PBS jobs for nested PVE/LXC/outer VMs; monthly PNY copies;
      quarterly restore drill (research/05, 28, 30).
- [ ] D2 Monitoring: host smartctl (wearout 99% alert!) + pve-exporter/
      Prometheus/Grafana inside PVE (research/07, 22, 30).
- [ ] D3 Security: TFA darko@pve (nested), libvirt network ACLs, Tailscale
      on host (research/16, 21), no :8006 exposure.
- [ ] D4 Power tuning Arrow Lake (research/15) - laptop battery matters.

## ProxLab (separate, stand-alone)

- [ ] P1 Convert X99 workstation to ProxLab PVE (research/31): data export,
      PVE on nvme0n1, desktop-as-VM w/ RX 5700 XT, rollback plan.
- [ ] P2 ProxLab storage (research/26): LVM-thin WD SSD, ZFS mirror NAS on
      2 HDDs, PNY detached PBS.
- [ ] P3 RX 5700 XT vendor-reset passthrough (research/25), D3cold fixes.
- [ ] P4 Standalone backups (research/28). No cluster with OmaLaptop.

## Security (cross-project)

- [x] Store darko@pve password + API token in Bitwarden (`OmaLaptop - Proxmox VE`)
- [ ] Move remaining hardcoded credentials in notes into Bitwarden placeholders
- [ ] TFA for the nested ProxDev PVE admin user (Phase D3)

## Done

- [x] Merge plan docs into repo under plan/ (2026-09-18)
- [x] PVE 9.2.20 bare-metal pilot on OmaLaptop (2026-09-17) - superseded by pivot
- [x] Pivot decision: Omarchy host + nested PVE recorded here and plan/omalaptop.md (2026-09-18)