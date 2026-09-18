---
tags: [convention, agents, setup]
created: 2026-09-17 20:39:30 +05
modified: 2026-09-18 09:35:08 +05
---

# AGENTS.md

> > IMPORTANT: Read **this file (AGENTS.md)** first to get the info and rules. Then refer to [docs/README.md](./docs/README.md) for the file inventory / index.
>
> Referenced by: [CLAUDE.md](./CLAUDE.md) - Claude and other agents must follow the conventions in that file.

## Purpose

Defines how AI agents (Claude, opencode, etc.) operate inside `~/Projects/Setup/`. Live notes live in `docs/` (index: [docs/README.md](./docs/README.md)), plan docs in `plan/` (index: [plan/README.md](./plan/README.md)), research notes in `research/` (index: [research/README.md](./research/README.md)).

## Mandatory conventions

Every note file (`.md`) in this directory MUST contain:

```yaml
---
tags: [tag1, tag2]
created: 2026-09-17 20:39:30 +05
modified: 2026-09-17 21:11:22 +05
---
```

Requirements (full detail in CLAUDE.md):

- `tags` - comma-separated category keywords inside a YAML list.
- `created` - file birth time, `YYYY-MM-DD HH:MM:SS TZ`. Never change after creation.
- `modified` - time of last edit, `YYYY-MM-DD HH:MM:SS TZ`. Update on EVERY edit.

## Agent rules

1. Read AGENTS.md (this file) first - it is the primary source of info.
2. See [docs/README.md](./docs/README.md) for the file inventory before working on docs.
3. Never run `git add .`/`git push` in this folder unless explicitly told - notes may contain credentials.
4. After editing any note, bump its `modified` timestamp to `date +"%Y-%m-%d %H:%M:%S %Z"`.
5. Do not create new notes without the frontmatter above.
6. Credentials-containing notes stay at `chmod 600`.
7. NO ICONS - no emojis/symbols/icons and no typographic/dash characters in notes. Only plain ASCII: use `-` (hyphen), `->` (arrow), `,` (comma). EXCEPTION: status indicators as plain uppercase tags: `[DANGER]`, `[CAUTION]`, `[WARN]`, `[INFO]`, `[OK]`.
8. USE DIAGRAMS - prefer Mermaid diagrams to describe flows, topologies, architecture. Use all kinds of markdown: mermaid diagrams (flowchart, sequence, class, state, gantt, pie, erDiagram, etc.), tables, lists, code blocks - wherever they add clarity.
9. ALWAYS UPDATE DOCS AFTER WORK - after completing any task, ALWAYS update the relevant docs: the affected note(s), [docs/tasks.md](./docs/tasks.md), [docs/changelog.md](./docs/changelog.md), and [docs/README.md](./docs/README.md) (file inventory/index if it changed). Never leave docs stale.