---
tags: [research, omalaptop, nested, proxmox, incus, lxc, omarchy, agent, mise, qemu]
created: 2026-09-18 17:58:10 +05
modified: 2026-09-18 18:25:03 +05
---

# Nested PVE VM vs flat virt + QEMU lab (with Omarchy reality)

Question under study: run the OmaLaptop lab tier inside a nested Proxmox VE VM,
or run everything "flat" with virt-manager/QEMU on the Omarchy host - given
that LXC/CT containers can also run directly on Arch. This note is the deep
research; it replaces the short answer recorded in changelog 2026-09-18 17:53.

## Framing

- Only the LAB TIER is in question. Outer VMs (GameWin11 dGPU, MainWin11) are
  virt-manager/QEMU no matter what - GPU passthrough only works at the outer
  level (hard rule, research/34).
- The host is Omarchy: an ARCHITECTED desktop, not vanilla Arch. That changes
  what "flat" means (section below).

## The host is Omarchy, and it is an agentic desktop (2026 facts)

- Full Omarchy / Omarchy-agent / mise deep dive lives in research/40. In one
  paragraph: Omarchy = Arch + Hyprland + Quickshell; `omarchy update` guards
  pacman; the agent lazy-installs agent CLIs (claude, codex, opencode, ...)
  via mise stubs in `~/.local/bin/` and owns `~/.agents/skills`
  (diagnose-crash, omarchy); mise is the primary runtime manager.
- Implication (kept here because it drives the verdict): OmaLaptop is a
  PERSONAL, agent-sculptable machine that updates on its own schedule
  (omarchy update + pacman -Syu). "Flat" would press it into double duty as
  the home server - updates then reboot the lab, containers run on the same
  btrfs as the desktop, and the agent helpers live on the same volatile box
  as the backups. That is the core tension nobody assigns a value to in the
  generic Proxmox-vs-Incus writeups.

## Arch-native containers: what "Arch can run LXC" really means (2026)

- Raw LXC: `lxc` package. No daemon, no API, snapshots only at the FS layer,
  config files by hand. Fine for a one-off; painful for a fleet of services.
  (bigiron.cc: nobody should hand-roll raw LXC configs in 2026 without a
  reason.) ArchWiki (Linux Containers) confirms the shape: privileged vs
  unprivileged (privileged = NOT root-safe), `/etc/subuid` + `/etc/subgid`
  with `lxc.idmap` lines in `/etc/lxc/default.conf` for unprivileged,
  `lxc-create -t download -B btrfs` for subvolume-backed rootfs, and - the
  practical killer - **overlayfs snapshots for UNPRIVILEGED containers are
  NOT supported in the mainline Arch kernel** (security). So even raw-LXC
  snapshot workflow is broken for the modern unprivileged setup on Arch.
- linuxcontainers.org/lxc/getting-started (the LXC project's own docs) is
  Ubuntu-flavoured (`apt-get install lxc`, `lxc-checkconfig`), stresses
  unprivileged as the only safe mode, and shows the manual wiring: lxcbr0 +
  dnsmasq DHCP leases, `LXC_DHCP_CONFILE` reservations, bind mounts by hand,
  `lxc.start.auto = 1`, and `systemd-run --user --scope -p "Delegate=yes"`
  wrappers so a non-root user can run containers under cgroup2. Everything
  self-managed, nothing policy-driven.
- **Incus** (recommended path) is the community fork of LXD, created after
  Canonical moved LXD behind a CLA in 2023. In Arch `extra` as `incus` /
  `incus-tools` (7.x), service `incus.socket`. Manages BOTH system containers
  and QEMU VMs with one CLI/daemon/API; storage pools (dir, btrfs, ZFS, LVM,
  ceph); instant COW snapshots/clones; `incus export` backups; managed bridge
  incusbr0; separate web UI (incus-ui). Migration: `lxd-to-incus`.
- LXD itself continues under Canonical; energy moved to Incus. Debian swapped
  LXD for Incus as default. ArchWiki LXD page is still Canonical-LXD shaped
  (package `lxd`, `lxd.socket`, client = the `lxc` binary, "anyone in the
  `lxd` group is root-equivalent", `lxd init`, `lxc launch ubuntu:20.04
  --vm`, VM boot needs `security.secureboot=false` because Arch ships no
  signed OVMF, and disk rw for unprivileged uses idmapped-mounts "shift" on
  the mainline kernel). It is the roadmap for what Incus does, packaged the
  Canonical way - another reason a flat lab = Incus, not LXD itself, here.
- virt-manager/libvirt does NOT fill this role well: its LXC driver is barely
  maintained and nobody uses it (bgnews24.org). So a "flat virt+QEMU lab"
  realistically = virt-manager for VMs + Incus (or raw lxc) for containers.

## LXC vs VM overhead (why density favors containers either way)

| Metric | LXC container | KVM VM (virtio) |
|---|---|---|
| Idle RAM (Debian) | 15-60 MB | 200-500 MB |
| Boot to ready | 0.5-4 s | 8-45 s |
| CPU vs bare metal | 99-100% | 97-99% |
| Disk IO vs bare metal | ~98% (native path) | 94-97% |
| Network vs bare metal | ~99% | ~97% |
| Backup image size | ~400 MB | ~2 GB |
| Windows / custom kernel | no (shared kernel) | yes |

Sources: proxmoxpulse, techfuelhq, wz-it. Conclusion: for a lab tier of
lightweight services, containers are the right form factor on EITHER
platform. The question is who manages them and their backups.

## Nested PVE reality (research/34 + 36 + new evidence)

- Prereq: outer CPU host-passthrough + `nested=1`; then the PVE guest gets
  its own /dev/kvm and depth-2 works. PVE wiki: CT (LXC) inside a nested
  guest is "quite usable" even WITHOUT hardware assist; with nested=1 it is
  comfortable. Depth-2 VM disk seq ~400-600 MB/s (vs ~2000 native).
- PVE gives the lab tier: pct, web UI (port 8006), RBAC/pveum, TFA,
  pve-firewall, vzdump + Proxmox Backup Server with incremental/dedup/
  encrypted/restore-drill workflows, REST API, Terraform/Ansible ecosystem -
  the exact skills later needed for ProxLab (research/31).
- Nested PVE is also the testbed: IncusOS and "PVE inside KVM for a lab" are
  mainstream practice (mattridpath, mrplanb) - nobody questions running PVE
  as a VM for learning/sandbox.

## Decision matrix (lab tier only; host = Omarchy)

| Criterion | Nested PVE VM (plan) | Flat: virt-manager + Incus on host |
|---|---|---|
| Mgmt surface (UI, API, RBAC, TFA, firewall) | built-in, complete | none built-in; assemble/DIY |
| Backups (vzdump + PBS, restore drills) | built-in, mature | incus export (full only) + restic/borg + PBS NOT packaged for Arch |
| Containers tooling | pct + templates + snapshots | Incus (excellent) |
| Container density | fine (CT usable depth-2) | best (host level) |
| Host update safety | lab isolated from host updates | updates reboot the lab with the desktop |
| Agent/Omarchy stack | stays host-local, uncluttered | entangled with lab workloads |
| Skills transfer to ProxLab (PVE) | full | none (ProxLab still PVE) |
| GPU passthrough flexibility | unchanged (outer only) | unchanged (outer only) |
| RAM cost | 16G fixed to PVE VM | ~0 |
| Depth-2 overhead | yes (VMs ~20-30% IO-hit; CT fine) | none |
| Double-NAT + S3 fragility | yes (research/34/36) | no |

## Verdict

Keep the nested PVE VM. The only genuinely strong "flat" points are RAM
savings and zero nesting overhead - but the lab tier is lightweight services
(AdGuard, PBS, monitoring, *arr) that tolerate depth-2 containers, and the
flat option costs the integrated backup/UI/RBAC/PBS story plus ProxLab
parity, while turning the agentic desktop into a server. The Arch-can-run-LXC
argument is answered by Incus existing, but Incus is a TOOL, not the
PLATFORM: it does not ship a backup scheduler, firewall, RBAC, TFA, or PBS,
and its good news (snapshots/exports) is exactly what PVE already combines
for both containers and VMs.

Given Omarchy's distro IS the desktop/agent environment, splitting
"personal machine" (host, Omarchy agent + mise + virt-manager outer VMs) from
"server part" (nested PVE VM, all the homelab machinery) is the cleanest
separation of concerns.

## Rules this implies (plan/omalaptop.md hard rules, 2026-09-18)

- Tiering escape hatch stays: IO-heavy software VMs may run OUTER (10.20.0.x)
  on virt-manager; only LXC + light VMs nested (research/36 numbers).
- Optional, later: install `incus` on the Omarchy host ONLY for throwaway
  containers, never for backed-up services - keeps backup story in PVE/PBS.
- PBS lives inside the nested lab tier (PBS-as-LXC, research/32); on Arch
  there is no official PBS package - another flat-path blocker.

## Sources

- omarchy.org/manual: AI (agent launchers + skills + diagnose-crash),
  development-tools (mise), omarchy-cli; basecamp/omarchy + omacom/omarchy
  releases (v4.0.0). Full omarchy/agent/mise note: research/40.
- archlinux.org: incus / incus-tools (extra); ArchWiki "Incus"; ArchWiki
  "Linux Containers" (raw LXC: subuid/subgid, idmaps, btrfs backing, no
  overlayfs snapshots unprivileged); ArchWiki "LXD" (lxd group root-eq,
  secureboot=false, shift/idmapped); linuxcontainers.org/lxc/getting-started.
- bigiron.cc LXC vs LXD vs Incus; bgnews24.org; homelabstarter, homenode,
  sumguy PVE-vs-Incus writeups; systhoughts (IncusOS note).
- proxmoxpulse, techfuelhq, wz-it LXC-vs-KVM measurements; PVE wiki
  Nested_Virtualization; pve-docs chapter-pct.
- Related notes: research/34 (nested), 35 (host stack), 36 (nested ops/perf),
  40 (Omarchy / agent / mise).

## Applied to / next steps

- Decision recorded; if user later wants a flat PoC, run the throwaway-VM A7
  test plus one Incus container in a weekend, then re-check this table.
- No task changes: Phase A/B roadmap already matches this verdict.