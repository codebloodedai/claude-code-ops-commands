---
name: planner-push
description: Push Claude's work (a % update + status note) back to Planner AND Pulse for a specific task
---

Write-back sync. Invoked when I (Claude) have just finished some work — a commit, a deploy, a bug fix, a vault update — and need to reflect it in Planner + Pulse immediately.

## Arguments (conversational — the user may supply these directly or I should infer them from current session context)
- **taskId** (required): the Planner task ID. If the user invokes /planner-push without args, ask which task — or propose the most likely one based on what we just worked on. If there's an anchor in `~/CLAUDE.md` for a bullet we just touched, use that taskId.
- **newPercent** (required): integer 0–100. If not provided, propose one based on the work done ("we just shipped X, I'd mark it Y%").
- **statusNote** (optional): a ≤200-char natural-language summary of what was done. Defaults to a one-liner inferred from the current session.

## Step 1: Confirm intent
Before any writes, show the user:
```
About to update:
  Task: <title>           (<taskId>)
  Plan: <plan name>
  %:    <old> → <new>
  Note: <statusNote>

This will write to:
  1. Planner (MS Graph)
  2. Pulse history (execdash PATCH)
  3. ~/CLAUDE.md bullet (the anchor that points to this task)

Proceed? [y/N]
```

Wait for explicit confirmation. **NEVER write without a yes.**

## Step 2: Fetch current Planner task state + etag
```
az rest --method GET --url "https://graph.microsoft.com/v1.0/planner/tasks/<taskId>"
```
Capture `.@odata.etag`. Fail out with a clear message if the task doesn't exist or I can't read it.

## Step 3: PATCH Planner percentComplete
```
az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/planner/tasks/<taskId>" \
  --headers "Content-Type=application/json" "If-Match=<etag>" \
  --body '{"percentComplete": <newPercent>}'
```
If the response is 204/200, continue. If 412 (etag mismatch), re-fetch and retry once. If still failing, stop and report.

## Step 4: Append a history entry to Pulse
Call the Pulse Functions API to upsert the project's history:
```
curl -X PATCH "https://<YOUR_DASHBOARD_FUNC_APP>/api/projects/<taskId>" \
  -H "Content-Type: application/json" \
  -d '{"history":[{"date":"<ISO-8601>","action":"<statusNote>","by":"<YOUR_EMAIL>"}]}'
```
**Caveat:** Pulse's function may require SWA-auth headers or managed-identity — if the direct PATCH returns 401/403, tell the user and suggest routing through the authenticated SWA instead (`curl https://<YOUR_DASHBOARD_DOMAIN>/api/projects/<taskId>`). Don't fail silently.

## Step 5: Update the bullet in ~/CLAUDE.md
Find the line in `~/CLAUDE.md` that contains `<!-- planner:<taskId> -->` and update its inline percentage with the Edit tool. Example:
```
old_string: - [ ] Ship monitoring integration `(You)` `10%` <!-- planner:<EXAMPLE_TASK_ID> -->
new_string: - [ ] Ship monitoring integration `(You)` `15%` <!-- planner:<EXAMPLE_TASK_ID> -->
```

If the task is now 100%, also flip `- [ ]` to `- [x]` and append `— *completed YYYY-MM-DD*` to the end of the bullet.

## Step 6: Report
Show a single-line confirmation of all three writes with their status codes. Example:
```
✅ Planner  204  <taskId>  percentComplete 10 → 15
✅ Pulse    200  history entry appended
✅ CLAUDE.md  line 87 updated
```

If any step failed, leave the others alone and show the user what got written vs what didn't so they can decide whether to manually reconcile.

## Constraints
- **Dry-run by default is NOT the behavior** — this command exists to commit state, not to preview it. Use /planner-pull for preview.
- Never skip the confirmation prompt in Step 1, even if the user passed all args.
- If the taskId has no anchor in ~/CLAUDE.md, Step 5 becomes "no matching anchor, skipping markdown update" — don't create a new bullet from nothing.
- If the session lacks the task's plan context (which bucket, which plan), fetch it from Planner first, don't guess.
