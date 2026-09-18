---
tags: [research, index, homelab, proxmox]
created: 2026-09-17 21:21:06 +05
modified: 2026-09-18 18:25:03 +05
---

# Research Index

Deep-research notes on homelab / Proxmox topics. Each note has its own tags and timestamps per [../AGENTS.md](../AGENTS.md).

## Notes (core)

- [01-homelab-architecture.md](./01-homelab-architecture.md) - Proxmox homelab architecture and best practices.
- [02-gpu-passthrough.md](./02-gpu-passthrough.md) - GPU passthrough with RTX 50-series for AI/LLM.
- [03-storage-zfs-vs-lvm.md](./03-storage-zfs-vs-lvm.md) - ZFS vs LVM-thin storage decisions.
- [04-networking.md](./04-networking.md) - VLANs, firewall, Tailscale for homelab.
- [05-backup-pbs.md](./05-backup-pbs.md) - Backup strategy with Proxmox Backup Server.
- [06-security-hardening.md](./06-security-hardening.md) - Proxmox security hardening.
- [07-monitoring.md](./07-monitoring.md) - Monitoring with pve-exporter, Grafana, Prometheus.

## Notes (platform / ops)

- [08-proxmox-cli.md](./08-proxmox-cli.md) - Proxmox CLI cheatsheet (qm, pct, pvesh, pveum, pveam, pvesm).
- [10-lxc-containers.md](./10-lxc-containers.md) - LXC containers: pct workflow, unprivileged, networking, backup.
- [13-iac-templates.md](./13-iac-templates.md) - Terraform/OpenTofu (bpg/proxmox), Ansible, cloud-init templates.
- [14-proxmox-api.md](./14-proxmox-api.md) - Proxmox REST API automation (curl/pvesh/tokens).
- [19-zfs-deep-dive.md](./19-zfs-deep-dive.md) - ZFS layout, tuning, snapshots, syncoid, migration.
- [23-docker-patterns.md](./23-docker-patterns.md) - Docker on Proxmox: LXC vs VM, Portainer/Dockge, OCI.

## Notes (apps / services)

- [09-homelab-services.md](./09-homelab-services.md) - Popular homelab services catalog with deploy advice.
- [11-community-scripts.md](./11-community-scripts.md) - community-scripts.org helper scripts (install, safety).
- [12-ai-stack.md](./12-ai-stack.md) - Ollama + Open WebUI on RTX 5060 8GB, models, tuning.
- [20-media-nas-stack.md](./20-media-nas-stack.md) - *arr media stack + gluetun + SMB/NFS sharing.

## Notes (network / access)

- [16-tailscale.md](./16-tailscale.md) - Tailscale subnet routers, serve, ACLs for remote access.
- [17-reverse-proxy-tls.md](./17-reverse-proxy-tls.md) - Caddy/Traefik/NPM, ACME, Authelia auth.
- [18-virtual-router-opnsense.md](./18-virtual-router-opnsense.md) - OPNsense VM, VLAN router-on-a-stick, NIC pairing.
- [33-pve-host-on-wifi.md](./33-pve-host-on-wifi.md) - Proxmox host on Wi-Fi: no wlan bridging, NAT/parprouted/routed options, roaming watchdog, PVE 9 wifi facts.
- [34-nested-pve-under-kvm.md](./34-nested-pve-under-kvm.md) - Nested Proxmox VE under a KVM desktop host: nested=1, host-passthrough CPU, thin qcow2, double-NAT, hard "no GPU into nested" rule.

## Notes (OmaLaptop pivot deep research, 2026-09-18)

- [35-outer-kvm-stack-omarchy.md](./35-outer-kvm-stack-omarchy.md) - Outer KVM host stack on Omarchy: qemu-desktop/libvirt/OVMF/swtpm, LUKS+btrfs+Limine reality, systemd units + hooks, VM XML + CPU pinning.
- [36-nested-pve-ops.md](./36-nested-pve-ops.md) - Nested PVE operations: host-passthrough + nested=1 prereqs, depth-1 overhead numbers, ext4 not ZFS, no passthrough inside, clock + snapshot ops.
- [37-arrow-lake-gfx-realities.md](./37-arrow-lake-gfx-realities.md) - Arrow Lake iGPU 7d67: GVT-g dead, SR-IOV VF broken (VFIO_MAP_DMA -EINVAL), concurrent-passthrough lockup risk; stays host-only.
- [38-rtx5060-blackwell-vfio.md](./38-rtx5060-blackwell-vfio.md) - RTX 5060 Blackwell VFIO: reset broken (one VM per boot), NO iommu=pt (1:1 IOMMU reject), vfio-pci.disable_idle_d3=1 + udev D0, GOP update.
- [39-nested-pve-vs-flat-virt-qemu.md](./39-nested-pve-vs-flat-virt-qemu.md) - Nested PVE vs flat virt+QEMU lab: Omarchy host reality, Incus (LXD fork) for Arch-native LXC, LXC-vs-VM overhead numbers, decision matrix -> nested PVE stays.
- [40-omarchy-agent-mise.md](./40-omarchy-agent-mise.md) - Omarchy distro, the Omarchy agent (mise stubs in ~/.local/bin, default agent, diagnose-crash + omarchy skills), and mise as runtime manager; the OmaLaptop host reality.

## Notes (security / reliability)

- [15-power-efficiency.md](./15-power-efficiency.md) - Arrow Lake 275HX idle power, C-states, ASPM tuning.
- [21-tfa-sso.md](./21-tfa-sso.md) - PVE TFA, Authelia/Authentik SSO, SSH 2FA.
- [22-monitoring-stack.md](./22-monitoring-stack.md) - Prometheus + pve-exporter + Grafana + Alertmanager/ntfy.

## Notes (ProxLab / platform, 2026-09-18)

- [24-dual-socket-x99-pve.md](./24-dual-socket-x99-pve.md) - X99 dual-socket (Broadwell-EP) platform for PVE: NUMA, ECC, IOMMU/PLX groups, power, watchdog.
- [25-amd-rx5700xt-passthrough.md](./25-amd-rx5700xt-passthrough.md) - RX 5700 XT (Navi 10) VFIO reset bug, vendor-reset DKMS for kernels >= 6.12, D3cold + hookscripts.
- [26-proxlab-storage-nas.md](./26-proxlab-storage-nas.md) - ProxLab storage build: NVMe root, WD SSD LVM-thin VM pool, ZFS mirror NAS on the 2 HDDs, PNY USB detached backup.
- [27-proxmox-cluster-vs-standalone.md](./27-proxmox-cluster-vs-standalone.md) - 2-node cluster + QDevice vs standalone for OmaLaptop/ProxLab; HA, fencing, ZFS replication, PVE 9 node-affinity.
- [28-offsite-backup-321.md](./28-offsite-backup-321.md) - 3-2-1 backup across both hosts: PBS + detached drive + encrypted offsite (B2/R2/rsync.net).
- [29-windows-gpu-vm.md](./29-windows-gpu-vm.md) - Windows 11 + gaming VMs: vTPM/Secure Boot, passthrough stack, HAGS/ReBar, anti-cheat reality (GameWin11/MainWin11/ProxLab desktop).
- [30-nvme-health-wearout.md](./30-nvme-health-wearout.md) - NVMe health/wearout on the hypervisors (OmaLaptop 96%/99%), smartctl monitoring, write-amplification cuts, replacement plan.
- [31-workstation-to-pve.md](./31-workstation-to-pve.md) - Migrating the omarchy workstation to ProxLab PVE safely: data export, desktop-as-VM, rollback/bare-metal fallback.
- [32-pbs-operations.md](./32-pbs-operations.md) - Running PBS as an LXC on ProxLab: install, datastore placement, schedules, retention/space math, encryption + restore drills.

## Applied to OmaLaptop

- OmaLaptop: Intel Core Ultra 9 275HX, 62.2 GB RAM, RTX 5060 Max-Q, 2x NVMe. See [../docs/omalaptop-hardware.md](../docs/omalaptop-hardware.md).
- Research date: 2026-09-17.

## Applied to ProxLab

- ProxLab: X99 dual-Xeon E5-2660 v4, 109 GB DDR4 ECC, RX 5700 XT, Lexar NVMe + 3x 1T SATA + PNY USB. See [../plan/proxlab.md](../plan/proxlab.md).
- Research date: 2026-09-18.