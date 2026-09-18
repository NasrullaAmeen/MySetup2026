---
tags: [research, omarchy, agent, mise, omalaptop, desktop]
created: 2026-09-18 18:02:00 +05
modified: 2026-09-18 18:25:03 +05
---

# Omarchy, the Omarchy agent, and mise (host reality, 2026)

Dedicated note for the host side of the OmaLaptop pivot: what Omarchy is, how its
agent integration works, and how mise drives both the agent launchers and the
18+ dev toolchains. Pulled out of research/39 so the flat-vs-nested decision
matrix stays about the lab, while this note owns the desktop facts.

## Omarchy the distro

- Omarchy (DHH, 37signals, MIT) = Arch base + Hyprland + Quickshell. v4
  "Quattro" (2026-08-14) rebuilt the shell as one plugin process.
- GPL/ALPM guarded updates: `omarchy update` (bypass with
  OMARCHY_ALLOW_DIRECT_PACMAN=1). Kernel on this box: 7.2.5-3-omarchy.
- Omarchy version installed on the ProxLab workstation today: `4.0.4-1`.
- Install mandates LUKS + btrfs (+Snapper) + Limine. Kernel cmdline lives in
  `boot/limine.conf`, NOT GRUB. Consequences for the OmaLaptop plan recorded in
  plan/omalaptop.md (Phase A kernel wrangling must target Limine, not GRUB).
- Config is user-owned: everything lives in `~/.config/` (hypr, omarchy,
  alacritty/foot/kitty/ghostty, btop, fastfetch, lazygit, starship, git).
  `/usr/share/omarchy/` is read-only package content (bin, config, themes,
  default, shell, migrations, install) - never edit, always read.
- CLI: single `omarchy` dispatcher over `omarchy-*` binaries. Groups: refresh,
  restart, toggle, theme, bar, plugin, hook, install, launch, capture,
  reminder, pkg, setup, update. Debug: `omarchy debug --no-sudo --print`.
- Safe edit flow: edit `~/.config/...`, validate Hyprland with `hyprctl reload
  + configerrors`, shell.json + user plugins hot-reload on save. Refresh backs
  up first: `omarchy refresh shell`. Nuclear: `omarchy reinstall`.

## The Omarchy agent (first-class citizen)

- **Agent launchers**: mise-managed stubs in `~/.local/bin/`. On the ProxLab
  workstation today: claude, codex, copilot, crush, cursor-agent, gemini, gh,
  ghui, grok, hermes, hunk, muse, omp, opencode, orca, orca-ide, pi,
  playwright, uv, uvx (source: `ls ~/.local/bin`). All tools self-install on
  first run via `mise use -g`.
- **Default agent**: chosen with `omarchy default agent`; launched with
  Super+Shift+Ctrl+A / `a`, started in `~/Work`. `omarchy agent` (with
  `--inline` / `--pick` variants) and `omarchy agent prompt <prompt...>` launch
  it; `omarchy agent usage update` regenerates usage dat files.
- **Skills**: shipped system skills symlinked into `~/.agents/skills` (and
  per-agent skill dirs). This box: `diagnose-crash` and `omarchy` are present.
  The `diagnose-crash` skill turns a systemd-coredump desktop toast into a
  `omarchy agent crash <pid>` workflow (coredumpctl -> debuginfod
  symbolization -> honest incident report). The `omarchy` skill brackets all
  end-user desktop config edits (hypr, shell.json, bar, themes, plugins).
- **The agent as integration point**: the agent edits real machine config, so
  any tooling we add for OmaLaptop must be agent-sculptable (files + omarchy
  CLI + skills), which is what the LXC/Incus decision must respect.
- Context: this whole repo session runs inside exactly this stack.

## mise (runtime manager)

- Omarchy's primary runtime manager. Today: `mise 2026.9.9` (linux-x64).
- 18+ dev environments install via `mise use -g <lang>`: ruby, node, bun,
  deno, go, python, java, elixir, dotnet, zig, clojure, scala. PHP via pacman,
  Rust via rustup, OCaml via opam; Python also uses uv.
- Config: `~/.config/mise/config.toml`. Installed right now: claude 2.1.274,
  codex 0.154.0, gh 2.101.0, node 26.8.1, opencode 1.18.31 (all "latest").
- The stubs set `MISE_MINIMUM_RELEASE_AGE=0` then `mise use -g --quiet` +
  `exec mise x <tool>` - lazy install, then direct exec.
- During `omarchy update` mise also refreshes agent launcher stubs + gh.

## Implications for OmaLaptop (laptop, after A1 Omarchy install)

- The laptop becomes the same kind of agentic desktop: mise stubs for all CLIs,
  `omarchy agent`, `omarchy` skill + `diagnose-crash` skill, kernels after
  `omarchy update`, Limine cmdline (Phase A steps must target Limine).
- It will NOT be a vanilla-Arch KVM box: updates flow through `omarchy update`;
  GPU/VM work must not fight the ALPM guard; the agent-accessible config
  surface is the real, supported way to shape the machine.

## Sources

- omarchy.org/manual: AI (agent launchers + skills + diagnose-crash),
  development-tools (mise), omarchy-cli; basecamp/omarchy + omacom/omarchy
  releases (v4.0.0, v4.0.4).
- Live capture 2026-09-18 18:01: omarchy version, kernel, ~/.local/bin stubs,
  ~/.agents/skills, mise --version, mise ls, omarchy commands.
- Related notes: research/39 (nested PVE vs flat verdict), 31 (workstation to
  PVE), 35 (outer KVM stack).