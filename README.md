# Claude Code Ops Commands

A set of slash commands for Claude Code that sync your **task tracker** (Microsoft Planner), **exec dashboard**, **personal vault** (markdown wiki), and **local CLAUDE.md** file — from one terminal session.

## What's included

```
commands/
├── checkpoint.md       Save everything everywhere — one confirmation
├── planner-link.md     One-time: match bullets to tasks, embed anchors
├── planner-pull.md     Read-only: tracker → CLAUDE.md diff
├── planner-push.md     Write-back: one task's % → tracker + CLAUDE.md
├── pulse-update.md     Interactive brief editor for an exec dashboard
├── status.md           Cross-project recon (tracker + vault + git)
├── today.md            Start/continue daily log
└── vault.md            Show vault index + flag stale notes

CLAUDE.md.example       Example ops command file with anchor pattern
```

## Setup

1. Copy the commands to your Claude Code config:
   ```bash
   cp commands/*.md ~/.claude/commands/
   ```

2. Copy the example and customize:
   ```bash
   cp CLAUDE.md.example ~/CLAUDE.md
   # Edit: add your team, objectives, Planner plan IDs
   ```

3. Replace placeholders in the command files:
   - `<YOUR_PLAN_ID_1>`, `<YOUR_PLAN_ID_2>`, `<YOUR_PLAN_ID_3>` → your Microsoft Planner plan IDs
   - `<YOUR_DASHBOARD_FUNC_APP>` → your dashboard's Azure Function App hostname
   - `<YOUR_DASHBOARD_DOMAIN>` → your dashboard's custom domain
   - `<YOUR_EMAIL>` → your email address

4. Bootstrap anchors:
   ```bash
   claude
   > /planner-link
   ```
   This matches your CLAUDE.md bullets to Planner tasks and embeds stable `<!-- planner:TASK_ID -->` join keys.

## How it works

### The anchor pattern

Every strategy bullet in your CLAUDE.md gets an invisible HTML comment with its Planner task ID:

```markdown
- [ ] Ship monitoring `(You)` `10%` <!-- planner:<EXAMPLE_TASK_ID> -->
```

This is the stable join key. Title-matching is fragile — anchors survive edits, renames, and rewordings.

### Daily workflow

| When | Command | What it does |
|------|---------|-------------|
| Session start | `/status` | Cross-project recon: stale vault, dirty git, tracker drift |
| Session start | `/planner-pull` | Sync tracker % → CLAUDE.md (read-only) |
| After shipping | `/planner-push` | Push new % to tracker + CLAUDE.md |
| After shipping | `/pulse-update` | Update exec dashboard brief (actions, risks, next steps) |
| End of session | `/checkpoint` | Save everything everywhere in one batch |

### Design principles

1. **One confirmation gate** — nothing writes silently
2. **Stable join keys** — HTML comment anchors, not title matching
3. **Zero new infrastructure** — markdown files + `az rest` + existing APIs
4. **Per-target failure isolation** — if the dashboard is unreachable, the vault and tracker still write

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- `az` CLI authenticated (`az login`)
- Microsoft Planner with tasks assigned to you
- A markdown vault (any local folder structure works)
- Optional: an exec dashboard with a PATCH API for briefs

## Customization

Each command is 30–80 lines of markdown. They're prompts, not scripts — edit them freely to match your tool stack. The pattern works with any task tracker that has a REST API, not just Planner.

### Adding per-project context loaders

Create one file per active project in `~/.claude/commands/`:

```markdown
---
name: myproject
description: Load MyProject context
---

Read `~/Code/myproject/CLAUDE.md` and `~/vault/projects/myproject.md`.
Summarize current state + top 3 open items.
Use ~/Code/myproject as the working directory for tool calls.
```

Then type `/myproject` to context-switch without leaving your home session.

## License

MIT — use it, modify it, share it.
