# Changelog

All notable changes to this repository are documented here.

## 2026-09-23 (rename + detach)

- Renamed the nested desktop-config repo `Omarchy-setup-2026/` -> `OmaLaptop/`;
  removed its `origin` remote (now local-only; the GitHub repos are left
  as-is, just unlinked). Dropped its public-mirror sync workflow and fixed
  internal name/path references.

## 2026-09-18 (pivot + rename)

- Pivoted: no bare-metal PVE on the laptop. OmaLaptop = Omarchy host (PHN16S-71);
  **ProxDev** = the nested PVE VM inside it (10.20.0.30, :8006 via host DNAT).
- Renamed files: `plan/proxdev.md` -> `plan/omalaptop.md`, `docs/proxdev-notes.md`
  -> `docs/omalaptop-notes.md`, `docs/proxdev-hardware.md` ->
  `docs/omalaptop-hardware.md`. PVE node id = `prodev` (`/nodes/prodev`).

## 2026-09-17

- Added all-rights-reserved `LICENSE`.
- Added `.gitignore`.
- Added `README.md` with project overview.
- Initial commit: `plan/hardware.md`, `plan/networking.md`, `plan/omalaptop.md`,
  `plan/proxlab.md`.

## 2026-09-17 (merge)

- Merged the Setup workspace (docs/, research/, AGENTS.md/CLAUDE.md) into this
  repo.
- Standardized the machine name to **OmaLaptop** everywhere in prose; the PVE
  node id stays lowercase `prodev` in API usage.
- Moved MySetup plan docs into `plan/`; added `plan/README.md` index; rewrote
  `README.md` to link plan/, docs/, research/.
- Plan docs updated to reality: OmaLaptop MGMT is now 10.10.10.10 on
  10.10.10.0/24 (planned 10.10.20.0/24 dropped); PVE 9.2.20 installed on
  nvme0n1; VM pool on nvme1n1 (LVM-thin).