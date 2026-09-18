---
tags: [changelog, docs]
created: 2026-09-17 20:53:22 +05
modified: 2026-09-18 18:25:03 +05
---

# Changelog

## 2026-09-18 18:30 - Rename: ProxDev = the nested PVE VM inside OmaLaptop

- Final naming (user decision): **OmaLaptop** = the PHN16S-71 host laptop
  (Omarchy desktop + virt-manager). **ProxDev** = the nested PVE VM inside
  OmaLaptop (10.20.0.30, :8006), formerly "OmaLaptop server".
- Renamed files: plan/proxdev.md -> plan/omalaptop.md, docs/proxdev-notes.md
  -> docs/omalaptop-notes.md, docs/proxdev-hardware.md ->
  docs/omalaptop-hardware.md.
- Swept `ProxDev` -> `OmaLaptop` (host) and `OmaLaptop server` -> `ProxDev`
  (VM) across all notes; PVE node id = `prodev` (API `/nodes/prodev`).
- The "retired bare-metal PVE era" record and 17:53/12:45 changelog entries
  kept as historical; earlier 12:45 pivot wording updated where ambiguous.

## 2026-09-18 18:11 - Omarchy/agent/mise note + OmaLaptop notes rewrite

- Research 40 added: dedicated deep-research note for Omarchy (distro,
  Limine/LUKS/btrfs reality), Omarchy agent (mise stubs in ~/.local/bin,
  default agent, skills diagnose-crash + omarchy), and mise as runtime
  manager. Research index now 40 topics.
- research/39 slimmed: "host is Omarchy" section replaced with pointer to
  research/40; Arch-native containers section expanded with verified facts
  from the 3 user-provided URLs (linuxcontainers.org/lxc/getting-started,
  ArchWiki Linux Containers, ArchWiki LXD) and a cleaner sources list.
- docs/omalaptop-notes.md replaced: Omarchy-era machine note covering both
  identities of the PHN16S-71 laptop -- OmaLaptop (retired bare-metal PVE
  era) and OmaLaptop (Omarchy era, 10.10.10.10, nested PVE 10.20.0.30),
  including the agent stack section and credentials/API record for the nested
  PVE later.

## 2026-09-18 17:58 - Deep research: nested PVE vs flat virt+QEMU

- Research 39 added: full comparison of nested PVE VM vs flat virt+QEMU lab,
  with the Omarchy host reality (Omarchy agent + mise + distro update model),
  Incus/LXD/LXC-on-Arch facts, LXC-vs-VM overhead numbers, decision matrix.
- Verdict: nested PVE stays (integrated backup/UI/RBAC/PBS, ProxLab parity,
  host stays the clean agentic desktop); escape hatch + optional host Incus
  for throwaway containers recorded. Research index now 39 topics.

## 2026-09-18 17:53 - Lab tier confirmed + home mgmt IP

- Decision: nested PVE stays the lab tier (LXC, PBS/vzdump, small VMs); flat
  virt+QEMU-only rejected (re-implements backup/ACL/container tooling).
  Added "tiering escape hatch": IO-heavy software VMs run outer (10.20.0.x),
  only LXC + light VMs nested (research/36, plan/omalaptop.md hard rules).
- Omarchy laptop owns **10.10.10.10** (old PVE IP reused): DHCP-reserve on
  router (host is DHCP on Wi-Fi) + host DNAT `10.10.10.10:8006 ->
  10.20.0.30:8006` keeps the old mgmt URL. plan/networking.md updated.
- Aligned GameWin11/MainWin11 outer IPs (.11/.12) between plan/omalaptop.md
  and plan/networking.md; added nested PVE 10.20.0.30 row. Task A8 added.

## 2026-09-18 17:00 - OmaLaptop pivot deep research

- Research 35-38: outer Omarchy/KVM host stack (qemu-desktop, libvirt, OVMF,
  swtpm, LUKS+btrfs+Limine, hooks/pinning), nested PVE ops + perf, Arrow
  Lake iGPU 7d67 realities (GVT-g dead, SR-IOV VF broken - host-only), RTX
  5060 Blackwell VFIO pitfalls.
- Corrected plan + tasks: `iommu=pt` REMOVED everywhere (Blackwell rejects
  a 1:1 IOMMU promise under passthrough mode -> VM start fails, research/38);
  added `vfio-pci.disable_idle_d3=1` + udev D0 rule; "one GPU VM start per
  host boot" usage rule (Blackwell reset broken).
- research/02 updated with the iommu=pt correction + links to research/38.
- Research index now 38 topics (docs/README.md, research/README.md updated).

## 2026-09-18 12:45 - ARCHITECTURE PIVOT (OmaLaptop)

- OmaLaptop drops bare-metal PVE. New design: Omarchy desktop host on iGPU +
  virt-manager; nested ProxDev PVE VM (8c/16G, standalone, NO GPU
  inside) hosts the lab tier (LXC services, software VMs, snapshots/backups).
  GameWin11/MainWin11 = outer virt-manager VMs (RTX 5060 VFIO to GameWin11).
- Research 34 added (nested PVE under KVM + hard "no GPU into nested" rule).
- Rewritten: plan/omalaptop.md (architecture/runbook A-E), docs/tasks.md
  roadmap (A host / B nested PVE / C outer GPU / D reliability / P proxlab),
  docs/omalaptop-notes.md (pivot status). PVE 9.2.20 pilot retired.

## 2026-09-18 12:30

- plan/networking.md reconciled with the "portable headless PVE on Wi-Fi"
  runbook: SSH references now darko@pve (root SSH disabled), added section
  18 Wi-Fi reliability (linux-firmware, power_save off, watchdog) from
  research/33.

## 2026-09-18 12:15

- Research 33 added: Proxmox host on Wi-Fi (why wlan bridging fails, NAT
  vs routed vs parprouted vs 4addr/WDS, iwlwifi no-4addr, roaming watchdog,
  firmware/linux-firmware notes). Index updated (33 topics).

## 2026-09-18 11:30

- Restructured docs/tasks.md into an ordered one-by-one OmaLaptop roadmap:
  Phase A (repos, IOMMU, pool, NAT, tooling), B (AdGuard, Tailscale, PBS),
  C (MainArch, iGPU, GameWin11, MainWin11), D (dev tier), E (backup,
  monitoring, security, power, stretch). ProxLab + Security sections kept.

## 2026-09-18 11:00

- Research 29-32 added: Windows/gaming VM stack (vTPM/SB, anti-cheat),
  NVMe wearout management (OmaLaptop P41 99% / SN520 96%), workstation to PVE
  migration plan, PBS-as-LXC operations runbook. Index updated (32 topics).

## 2026-09-18 10:30

- Added docs/proxlab-notes.md (status/roadmap/access plan) and
  docs/proxlab-hardware.md (full X99 hardware stack). Updated docs/README.md
  index (28 research topics).

## 2026-09-18 10:15

- Research expanded for ProxLab: added 24-dual-socket-x99-pve,
  25-amd-rx5700xt-passthrough (vendor-reset, D3cold), 26-proxlab-storage-nas,
  27-proxmox-cluster-vs-standalone (QDevice, pvesr, node-affinity),
  28-offsite-backup-321. Index updated, "Applied to ProxLab" section added.

## 2026-09-18 09:53

- Captured the hardware stack of the X99 workstation (`omarchy`, 2x Xeon
  E5-2660 v4 / 28C-56T, 109G DDR4, RX 5700 XT, 4x 1TB + Lexar NVMe) into
  plan/proxlab.md - this PC will become **ProxLab**. IOMMU groups 83/84/85
  mapped for GPU passthrough.

## 2026-09-18

- Merged plan docs into `plan/` (hardware, networking, omalaptop, proxlab) with
  index [plan/README.md](../plan/README.md). Rewrote root `README.md` linking
  plan/, docs/, research/.
- Standardized machine name to **OmaLaptop** everywhere; PVE node id stays
  lowercase `prodev` in API usage (`/nodes/prodev`) - the node lives inside
  the nested ProxDev VM (2026-09-18, 18:2x).
- Plan docs updated to reality: OmaLaptop MGMT 10.10.10.10 (10.10.10.0/24),
  PVE 9.2.20 on nvme0n1, VM pool nvme1n1.
- Renamed notes to lowercase: `docs/omalaptop-notes.md`, `docs/omalaptop-hardware.md`.

## 2026-09-17 21:42

- Research expanded from 7 to 23 notes. Added: Proxmox CLI cheatsheet (08), homelab services catalog (09), LXC containers (10), community-scripts.org (11), AI stack Ollama/Open WebUI on RTX 5060 (12), IaC Terraform/Ansible/cloud-init (13), REST API automation (14), power efficiency Arrow Lake 275HX (15), Tailscale (16), reverse proxy + TLS + auth (17), OPNsense virtual router (18), ZFS deep-dive (19), media/NAS stack (20), TFA & SSO (21), monitoring stack (22), Docker patterns (23).
- Research/README.md reorganized by theme: core / platform-ops / apps-services / network-access / security-reliability.

## 2026-09-17 21:25

- Created research/ folder with 7 deep-research notes (homelab architecture, GPU passthrough RTX 50, ZFS vs LVM, networking, PBS backup, security hardening, monitoring). Index: research/README.md. See docs/README.md.
- Installed uv + pip on client for automation.

## 2026-09-17 21:19

- Added Mermaid network topology diagram + resource/access/storage tables to omalaptop-notes.md.
- Added hardware summary table to omalaptop-hardware.md.

## 2026-09-17 21:18

- AGENTS.md rule 9 added: always update docs after work (affected notes, tasks.md, changelog.md, README.md).

## 2026-09-17 21:12

- Disabled root SSH login on OmaLaptop: `PermitRootLogin no`, sshd reloaded, verified root login rejected. Backup config on host: `/etc/ssh/sshd_config.bak.20260917`.
- AGENTS.md rule 8 added: use Mermaid diagrams + all kinds of markdown (tables, lists, code).
- Installed uv + pip (pip via uv tool) on client for paramiko-based SSH automation.

## 2026-09-17 21:06

- AGENTS.md rule 7 expanded: banned all typographic symbols (em dashes, middle dots, arrows, etc.) - only plain ASCII. Status tags `[WARN]`/`[DANGER]` etc. still allowed.
- Cleaned all notes + AGENTS.md to plain ASCII.

## 2026-09-17 20:55

- Stored `darko@pve` password + API token in Bitwarden (item: `OmaLaptop - Proxmox VE`).
- Scrub credentials from notes -> replaced with Bitwarden placeholders/redacted.

## 2026-09-17

- Connected to OmaLaptop (Proxmox VE 9.2.20) at 10.10.10.10:8006.
- Gathered full hardware stack (see [omalaptop-hardware.md](./omalaptop-hardware.md)).
- Root hardening: created `darko@pve` (Administrator), disabled `root@pam`, created API token `darko@pve!clitoken`.
- Created Setup folder conventions: [AGENTS.md](../AGENTS.md) + [CLAUDE.md](../CLAUDE.md).
- Moved notes into `docs/` and created this changelog + tasks.

_Modified: bump this section's timestamp whenever editing this file._