---
tags: [changelog, docs]
created: 2026-09-17 20:53:22 +05
modified: 2026-09-18 09:40:00 +05
---

# Changelog

## 2026-09-18

- Merged plan docs into `plan/` (hardware, networking, proxdev, proxlab) with
  index [plan/README.md](../plan/README.md). Rewrote root `README.md` linking
  plan/, docs/, research/.
- Standardized machine name to **ProxDev** everywhere; PVE node id stays
  lowercase `prodev` in API usage (`/nodes/prodev`).
- Plan docs updated to reality: ProxDev MGMT 10.10.10.10 (10.10.10.0/24),
  PVE 9.2.20 on nvme0n1, VM pool nvme1n1.
- Renamed notes to lowercase: `docs/proxdev-notes.md`, `docs/proxdev-hardware.md`.

## 2026-09-17 21:42

- Research expanded from 7 to 23 notes. Added: Proxmox CLI cheatsheet (08), homelab services catalog (09), LXC containers (10), community-scripts.org (11), AI stack Ollama/Open WebUI on RTX 5060 (12), IaC Terraform/Ansible/cloud-init (13), REST API automation (14), power efficiency Arrow Lake 275HX (15), Tailscale (16), reverse proxy + TLS + auth (17), OPNsense virtual router (18), ZFS deep-dive (19), media/NAS stack (20), TFA & SSO (21), monitoring stack (22), Docker patterns (23).
- Research/README.md reorganized by theme: core / platform-ops / apps-services / network-access / security-reliability.

## 2026-09-17 21:25

- Created research/ folder with 7 deep-research notes (homelab architecture, GPU passthrough RTX 50, ZFS vs LVM, networking, PBS backup, security hardening, monitoring). Index: research/README.md. See docs/README.md.
- Installed uv + pip on client for automation.

## 2026-09-17 21:19

- Added Mermaid network topology diagram + resource/access/storage tables to proxdev-notes.md.
- Added hardware summary table to proxdev-hardware.md.

## 2026-09-17 21:18

- AGENTS.md rule 9 added: always update docs after work (affected notes, tasks.md, changelog.md, README.md).

## 2026-09-17 21:12

- Disabled root SSH login on ProxDev: `PermitRootLogin no`, sshd reloaded, verified root login rejected. Backup config on host: `/etc/ssh/sshd_config.bak.20260917`.
- AGENTS.md rule 8 added: use Mermaid diagrams + all kinds of markdown (tables, lists, code).
- Installed uv + pip (pip via uv tool) on client for paramiko-based SSH automation.

## 2026-09-17 21:06

- AGENTS.md rule 7 expanded: banned all typographic symbols (em dashes, middle dots, arrows, etc.) - only plain ASCII. Status tags `[WARN]`/`[DANGER]` etc. still allowed.
- Cleaned all notes + AGENTS.md to plain ASCII.

## 2026-09-17 20:55

- Stored `darko@pve` password + API token in Bitwarden (item: `ProxDev - Proxmox VE`).
- Scrub credentials from notes -> replaced with Bitwarden placeholders/redacted.

## 2026-09-17

- Connected to ProxDev (Proxmox VE 9.2.20) at 10.10.10.10:8006.
- Gathered full hardware stack (see [proxdev-hardware.md](./proxdev-hardware.md)).
- Root hardening: created `darko@pve` (Administrator), disabled `root@pam`, created API token `darko@pve!clitoken`.
- Created Setup folder conventions: [AGENTS.md](../AGENTS.md) + [CLAUDE.md](../CLAUDE.md).
- Moved notes into `docs/` and created this changelog + tasks.

_Modified: bump this section's timestamp whenever editing this file._