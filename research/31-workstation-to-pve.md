---
tags: [research, proxlab, migration, workstation, pve, desktop, vm]
created: 2026-09-18 11:00:00 +05
modified: 2026-09-18 18:25:03 +05
---

# 31. Turning this workstation (omarchy) into ProxLab PVE host

Research date: 2026-09-18. "This PC will become ProxLab" - but right now it
is the **daily-driver Omarchy desktop** (kernel 7.2.5-3-omarchy) this repo is
being developed on. The end state is Proxmox VE on nvme0n1 with everything
living as VMs/LXC. This note is the migration safety plan.

## The trap

A full reinstall of PVE wipes the only desktop this person uses. Risky
unless (a) data is exported, (b) the desktop comes back as a VM with the GPU
attached (research/25 + 29), and (c) there is a rollback path. Do NOT run
`pvemmenu` live on the boot disk before the export is verified.

## Phase 0 - inventory what must survive

| What | Where it lives | Export path |
|---|---|---|
| This repo + docs | ~/Projects/MySetup2026 (+ GitHub remote) | already safe (git + remote origin) |
| Dotfiles / shell history / SSH keys | /home/darko (nvme0n1 LUKS root) | `tar` + push to Git/Bitwarden or the PNY |
| Editors/configs (.config) | /home/darko/.config | same tar |
| Media / family files | /run/media/darko/Hdscythe (PNY) + any sda/sdb/sdc | keep on disks (not PVE yet) |
| Databases (if any) | - | dump + copy |

- Because the drives stay in the box, most things are safe *as long as we do
  NOT change partitioning of nvme0n1 before copying what is on it now*.

## Phase 1 - keep the desktop as a VM (dual purpose)

Two viable directions:

1. **Convert the current Omarchy install into a KVM guest** - laborious but
   keeps all configs: shrink/resize the LUKS ext4/btrfs root into an image
   (`virt-p2v` / `clonezilla` disk-to-image) then run it in PVE on the SATA
   SSD with the RX 5700 XT passed through. Restorable later; needs a second
   boot path for repair.
2. **Reinstall Omarchy (or Arch) as a clean PVE VM and re-setup** - faster,
   loses only tweaking time, this repo IS the notes so nothing is lost;
   recommended unless the existing install is heavily customized.

Do one, then PVE owns nvme0n1.

## Phase 2 - order of operations on the hardware

1. PVE on nvme0n1 (Lexar), as [proxlab-notes.md](../../docs/proxlab-notes.md)
   roadmap (use the omalaptop.md runbook).
2. Storage: WD SSD = LVM-thin VM pool; 2 Seagate = ZFS mirror NAS;
   PNY = detached PBS (research/26).
3. First guests: AdGuard LXC + desktop VM (RX 5700 XT passthrough w/
   vendor-reset, research/25).
4. Then bring over anything from the old desktop VM (dotfiles etc. are
   already exported in Phase 0).

## Phase 3 - rollback / bare-metal fallback

- Keep a **bootable USB** (Omarchy live) + an image of the old root
  (`clonezilla` image on the PNY) for 30 days after cutover.
- If GPU passthrough proves unusable and no mini-display exists: boot PVE
  headless with `nofb initcall_blacklist=sysfb_init`, admin from the laptop/
  router over SSH + :8006, and run a *software-rendered* desktop LXC until
  a cheap secondary GPU (e.g. a GT 710) arrives.
- Document `hostname omarchy -> proxlab` change (tasks list).

## Timeline posture

- Keep the workstation usable until the desktop VM boots from the pool with
  the GPU working. Comms during flips: use the laptop (OmaLaptop) + Tailscale.
- Accept ~1-2 evenings of work; keep a written runbook (this plan + tasks).

## Open questions

- Decide 1 vs 2 in Phase 1 (chancy image conversion vs clean reinstall) -
  default recommendation: **clean reinstall**, the box is a notes-driven
  homelab, not the keeper of years of tweaks.
- Budget a tiny second GPU (GT 710-class, ~$30) to keep a local console
  during bring-up - cheap insurance for passthrough experiments.