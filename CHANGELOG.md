# Changelog

All notable changes to this repository are documented here.

## 2026-09-17

- Added all-rights-reserved `LICENSE`.
- Added `.gitignore`.
- Added `README.md` with project overview.
- Initial commit: `plan/hardware.md`, `plan/networking.md`, `plan/proxdev.md`,
  `plan/proxlab.md`.

## 2026-09-17 (merge)

- Merged the Setup workspace (docs/, research/, AGENTS.md/CLAUDE.md) into this
  repo.
- Standardized the machine name to **ProxDev** everywhere in prose; the PVE
  node id stays lowercase `prodev` in API usage.
- Moved MySetup plan docs into `plan/`; added `plan/README.md` index; rewrote
  `README.md` to link plan/, docs/, research/.
- Plan docs updated to reality: ProxDev MGMT is now 10.10.10.10 on
  10.10.10.0/24 (planned 10.10.20.0/24 dropped); PVE 9.2.20 installed on
  nvme0n1; VM pool on nvme1n1 (LVM-thin).