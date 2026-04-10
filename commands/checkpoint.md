---
name: checkpoint
description: Save everything, everywhere — daily log, vault project notes, Planner %, Pulse briefs — based on what we did this session
---

One-shot session save. Run this when you want to capture the state of the current session into all four places at once:
1. `~/vault/daily/YYYY-MM-DD.md` — daily log (always)
2. `~/vault/projects/<project>.md` — vault project notes (one per project touched)
3. **Planner** — % complete via `az rest` PATCH
4. **Pulse** — brief + history via authenticated PATCH

This is a meta-command that orchestrates the existing /planner-push and /pulse-update flows plus the vault/daily-log writes. Respects the same safety rules: nothing gets written without explicit confirmation.

## Step 1: Detect projects touched in this session
Look at the conversation + recent file edits + git state to identify which projects were worked on. Signals:
- Files edited under `~/Code/<repo>/` → that project is touched
- Git commits created in this session → the repo's project is touched
- Slash commands invoked (`/myproject`, `/portal`, etc.) → those projects are touched
- Deploys that ran (`func publish`, `swa deploy`, `gh run watch`)
- Vault notes updated
- Azure CLI commands against a resource group that maps to a project

Build a set: `{ project_key: summary_of_work }`. The project_key is the short slug (e.g., `project-alpha`, `portal`, `hopper`, `cc`). If in doubt, ask the user which projects to include. **Don't guess wildly** — if work spans ambiguous boundaries, ask.

## Step 2: Show the detected scope + ask for confirmation
```
📦 Checkpoint scope (auto-detected):

  Project Alpha:
    - Fixed SWA linked-backend routing
    - Merged fix/csp-inline-handlers → main
    - Added DNS A record for Azure OpenAI PE
    - Backfilled references on all 4 records
    Signals: 5 commits in ~/Code/project-alpha, /project-alpha invoked,
             az rest against rg-project-alpha-prod, Playwright against SWA

  Process Portal:
    - Added CLAUDE.md stub
    Signals: 1 untracked file

Targets to update:
  [✓] ~/vault/daily/2026-04-09.md   (append session summary)
  [✓] ~/vault/projects/project-alpha.md (update state + risks + todos)
  [ ] ~/vault/projects/process-portal.md (skipping — CLAUDE.md only, no material change)
  [✓] Planner task: "Project Alpha" → 75 → 85
  [✓] Pulse brief: Project Alpha (actions, next steps, risks)

Proceed with all 4 writes? [y / edit scope / abort]
```

## Step 3: Daily log write
Append a section to `~/vault/daily/YYYY-MM-DD.md` with today's date (compute it, don't assume). Header format:
```
## <Project Name> — <one-line outcome>
```
Body: 3–6 bullets summarizing what was done, links to relevant commits, and flag any follow-ups.

**Don't create duplicate sections** — if today's log already has a section with the same project header, append to it with a sub-heading `### <HH:MM> additional work` instead of duplicating.

**Never include personnel/performance notes in the daily log.** Keep career-track discussions and behavioral assessments out — recommendations about policy/technical changes are fine.

## Step 4: Vault project note updates
For each project touched, read the existing `~/vault/projects/<slug>.md` and update the sections that have actually changed:
- **Status line** at top (e.g., "80% → 85% — audio playback working after 2026-04-08 recovery")
- **Session changelog section** — append a new dated entry `## Session YYYY-MM-DD Changes` with the work summary
- **Next Steps checklist** — mark completed items `[x]` and add new items discovered
- **Related** / **Known Issues** — update if new risks were uncovered

If a vault project note doesn't exist yet for a project we touched, flag it and ask if I should create a stub from the template.

Use the Edit tool for surgical updates, not Write. Never rewrite the whole file.

## Step 5: Planner updates (delegate to /planner-push logic)
For each project with a Planner anchor in `~/CLAUDE.md`:
1. Compute the new % based on the work (propose, don't invent)
2. Confirm the old → new number with the user
3. Fetch current task + etag, PATCH `percentComplete` via `az rest`
4. Update the anchored bullet in `~/CLAUDE.md`

Skip projects that have no anchor — flag them as "no planner link, not pushing" and continue.

If the user wants to mark something fully complete (100%), also flip `- [ ]` → `- [x]` in the CLAUDE.md bullet and add `— *completed YYYY-MM-DD*`.

## Step 6: Pulse updates (delegate to /pulse-update logic)
For each project with a matching Pulse entry (keyed by Planner task ID):
1. Fetch current brief from Pulse via the SWA path (`https://<YOUR_DASHBOARD_DOMAIN>/api/projects/<taskId>`). If no authenticated browser session is open, tell the user and SKIP Pulse for this project rather than failing the whole checkpoint.
2. Propose a single new history entry (append, never overwrite)
3. Propose updates to `nextActions`, `brief.risks`, `brief.currentReality` where they've materially changed
4. Ask per-project for confirmation of the brief payload (one round of questions per project, not per field — we're in batch mode, keep the cadence fast)
5. PATCH via SWA

**If Pulse auth is unavailable for any project, don't fail the whole run.** Write what you can (daily log, vault, Planner) and report which Pulse updates were skipped so the user can run `/pulse-update <project>` manually later.

## Step 7: Final report
Show a single status table at the end:

```
✅ Checkpoint complete (3 of 4 writes succeeded)

  Daily log   │ ✅ appended 1 section to ~/vault/daily/2026-04-09.md
  Vault       │ ✅ project-alpha.md updated (status + changelog + next steps)
  Planner     │ ✅ Project Alpha  75 → 85  (task abc123...)
              │ ⏭  Process Portal — no anchor, skipped
  Pulse       │ ⚠  Project Alpha — no authenticated SWA session, skipped
              │   → run: /pulse-update project-alpha  to retry manually
  CLAUDE.md   │ ✅ 1 bullet updated (anchor abc123... 75% → 85%)

Session covered ~3 hours of work across 1 primary + 1 incidental project.
```

If everything succeeded, be terse and move on. If anything failed or was skipped, make it obvious what needs manual follow-up.

## Constraints
- **One confirmation gate at the top.** After Step 2's `[y]`, the rest of the run should be minimally interactive — only ask about ambiguities (e.g., "what % should Project Alpha be?"). Don't stop for every field.
- **Never write without Step 2 confirmation.**
- **Never skip the daily log.** Even if no projects were touched, append a short "no material work" entry rather than leaving a gap — the daily log is the audit trail.
- **Never inject personnel details into the daily log.**
- **Never overwrite Pulse `history`** — always fetch + append.
- **Never touch Planner-managed fields** (title, dueDate, assignees). Only `percentComplete`.
- **Fail gracefully per target.** If Pulse is down, Planner and vault should still write. If Planner PATCH fails, the vault should still write. Each target is independent; one failure doesn't abort the others.
- **Report honestly at the end.** If Pulse was skipped, say so. If a project had no anchor, say so. Never claim a write landed that didn't.
- **Sync vault to OneDrive at the very end** by running `bash ~/vault/.wiki/sync.sh` so SharePoint is fresh.
