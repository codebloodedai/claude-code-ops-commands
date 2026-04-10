---
name: planner-pull
description: Pull your assigned tasks from Planner, diff against ~/CLAUDE.md, optionally update the markdown
---

Read-only-by-default Planner sync. Do the following:

## Step 1: Get your Planner user ID
Run `az rest --method GET --url "https://graph.microsoft.com/v1.0/me?\$select=id,displayName,mail"` and capture `.id`. Cache it in a bash variable for this session.

## Step 2: Pull tasks from all 3 active plans in parallel
Plan IDs (authoritative list — source of truth is `~/CLAUDE.md` "Microsoft Planner Reference"):
- **IT Operations**: `<YOUR_PLAN_ID_1>`
- **Enterprise Engineering & Operations**: `<YOUR_PLAN_ID_2>`
- **AI Initiatives**: `<YOUR_PLAN_ID_3>`

For each plan, run:
```
az rest --method GET --url "https://graph.microsoft.com/v1.0/planner/plans/<planId>/tasks"
```

## Step 3: Filter to tasks assigned to you
From each plan's `value` array, keep tasks where `assignments` contains your user ID as a key. Grab per-task: `id`, `title`, `percentComplete`, `dueDateTime`, `bucketId`, `@odata.etag`.

## Step 4: Parse ~/CLAUDE.md for existing anchors
Read `~/CLAUDE.md`. For every strategy bullet that already has an HTML anchor of the shape `<!-- planner:<taskId> -->`, build a map `{ taskId → { line_number, current_pct_from_bullet, title_from_bullet } }`.

## Step 5: Build the delta report
Group tasks by plan. For each task, classify as:
- **✅ synced** — has anchor in CLAUDE.md AND `%Complete` matches (tolerance: exact integer match)
- **🟡 drift** — has anchor but `%Complete` differs (Planner authoritative)
- **🔴 unlinked** — not in CLAUDE.md yet (no anchor), probably needs adding to a strategy
- **⚫ orphan** — anchor in CLAUDE.md but task no longer exists in Planner (closed/deleted)

Display the report as a clean table, one section per plan, with these columns:
| Status | Title | Planner % | CLAUDE.md % | Due | Bucket |

## Step 6: Propose updates
After the table, list the proposed changes to `~/CLAUDE.md`:
- For each 🟡 drift: show before/after of the bullet line
- For each 🔴 unlinked: suggest which OBJ/Strategy section it would fit under based on title similarity (don't auto-insert; just propose)
- For each ⚫ orphan: suggest removing the anchor and marking the bullet with `<!-- planner:closed -->`

**Do not modify ~/CLAUDE.md until the user explicitly confirms.** Ask: "Apply these N changes to ~/CLAUDE.md? [y/N]"

## Step 7: Apply (if confirmed)
Use the Edit tool with surgical replacements — one `old_string → new_string` per drift. Never rewrite the whole file. For each change, the `old_string` should be the complete bullet line including the anchor, and `new_string` should be the same line with the updated `XX%` value.

## Constraints
- This is READ-ONLY against Planner (no PATCH). Only writes to ~/CLAUDE.md after explicit confirmation.
- Never modify any bullet that lacks an anchor — it might be intentionally manual or on a different system.
- If `az rest` returns a 401/403, tell the user to run `az login` and stop.
- If ~/CLAUDE.md has no anchors yet, print a short "no anchors yet — run /planner-link to bootstrap" message and the full task list so they can manually embed anchors.
