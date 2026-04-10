---
name: pulse-update
description: Update a Pulse project brief (actions taken, next steps, risks, current reality) based on the work we just did in this session
---

Interactive editor for Pulse project briefs. Used AFTER finishing a chunk of work to reflect it in the exec dashboard. Distinct from /planner-push — this command is editorial (brief/narrative content), /planner-push is mechanical (% complete).

## Pulse brief data shape

Pulse projects are keyed by their **Planner task ID** and stored in Azure Table Storage. Each project has these dashboard-only editable fields (set via `PATCH /api/projects/{taskId}`):

| Field | Type | Semantics |
|---|---|---|
| `brief.objective` | string | What the project is trying to achieve. Rarely changes. |
| `brief.currentReality` | string | Where we are today. Should reflect the most recent material change. |
| `brief.risks` | string[] | Active risks/blockers. Add new ones, remove resolved ones. |
| `nextActions` | `{ action, owner, dueDate }[]` | Forward-looking work items. Replaces fully on write. |
| `history` | `{ date, action, by }[]` | Append-only feed of what has been done. |
| `priority` | int | Dashboard ordering hint. Don't touch unless explicitly asked. |

The Planner-managed fields (`title`, `percentComplete`, `dueDate`, `assignees`) flow FROM Planner and should NOT be touched here — use /planner-push for those.

## Step 1: Identify which project we're updating
If the user invokes /pulse-update without args, determine the target:
- If the last ~20 turns of conversation are clearly about one project (e.g., we just fixed project audio, we just committed a backend feature), propose that project and its Planner task ID.
- If there's no clear context, list the open projects from `/api/projects` and ask which one.
- If the user passes a name ("pulse-update project-alpha") or a Planner task ID, use that directly.
- Prefer resolving via `<!-- planner:<taskId> -->` anchors in `~/CLAUDE.md` — they're the deterministic join key.

## Step 2: Fetch the current brief
Hit Pulse's GET endpoint to retrieve the current state of the project. Two paths:

**Preferred: via authenticated SWA (works when a browser session is available)**
```
curl https://<YOUR_DASHBOARD_DOMAIN>/api/projects
```
Requires an MSAL session. If we have Playwright already open on a <YOUR_DASHBOARD_DOMAIN> tab, reuse it via `browser_evaluate` to fetch the JSON.

**Fallback: direct function app (will 401 without SWA auth header)**
```
curl https://<YOUR_DASHBOARD_FUNC_APP>/api/projects
```
This will fail from my laptop but works if we're running inside the SWA-linked context.

If both paths fail, tell the user: "I can't reach Pulse without an authenticated session. Open <YOUR_DASHBOARD_DOMAIN> in Playwright first, then re-run /pulse-update." Stop.

Once fetched, filter to the target project by Planner task ID.

## Step 3: Display the current brief
Show the user:
```
📊 Pulse brief: <title>  (taskId: <id>)

  Objective:       <one-line objective>
  Current Reality: <one-line current state>
  Risks:
    1. <risk 1>
    2. <risk 2>
    ...
  Next Actions:
    1. <action 1> — <owner>, due <date>
    2. <action 2> — <owner>, due <date>
  History (last 5 entries):
    2026-04-08  <you> <action>
    2026-04-03  <you> <action>
    ...
```

## Step 4: Propose updates based on the session
Review the conversation history — what did we actually do? Based on the work, propose one update per relevant field:

**Actions Taken** (append to history) — REQUIRED for every /pulse-update invocation:
A one-sentence description of the most recent material action. Format: `YYYY-MM-DD: <one-sentence action>`. Examples:
- "Fixed SWA linked-backend routing after April 3 storage lockdown — unlinked/relinked, restored Entra auth path"
- "Shipped deliverable upload widget in Work Queue + user workflow guide"
- "Backfilled `transcriptRef` on all 4 scored calls — rubric play buttons now render"

**Next Steps** (overwrite `nextActions` array) — USUALLY updates:
Rewrite the forward-looking list. Remove items we just completed. Add items that became obvious during this session. Keep existing open items that still apply. Items are: `{ action, owner, dueDate }`. If `owner` or `dueDate` is unknown, use `null` — don't make them up.

**Risks** (overwrite `brief.risks` array) — SOMETIMES updates:
Add any risks we uncovered. Remove any risks we resolved. Keep others. A risk is a short sentence, ≤100 chars, starting with the problem, not the solution.

**Current Reality** (overwrite `brief.currentReality`) — OCCASIONALLY updates:
If the session materially changed what's true about the project RIGHT NOW, propose a rewrite. Otherwise leave it alone.

**Objective** (overwrite `brief.objective`) — RARELY updates:
Only propose changes if the project's goal has fundamentally shifted. 99% of the time, don't touch.

## Step 5: Interactive confirmation per field
For EACH proposed change, show a clean before/after and ask: "Keep / edit / skip?"

```
🆕 Add to History:
   2026-04-09: Merged fix/csp-inline-handlers to main, fixed V8
               context.log binding, added PE DNS record for Azure
               OpenAI, backfilled transcriptRef on 4 scored calls.

   [k]eep  [e]dit  [s]kip  ?
```

```
✏️  Update Next Steps:
   BEFORE:
     1. Debug audio playback in browser console
     2. Calibrate scoring with QA supervisor
     3. Score 15+ more PNC calls for calibration

   AFTER:
     1. Migrate STORAGE_CONNECTION_STRING to KV reference (you)
     2. Rotate vendor API private key (you)
     3. Score 15+ more PNC calls for calibration
     4. Calibrate scoring with QA supervisor

   [k]eep  [e]dit  [s]kip  ?
```

```
⚠️  Update Risks:
   BEFORE:
     - Audio playback blocked (known issue since 2026-03-23)
     - STORAGE_CONNECTION_STRING plaintext in app settings

   AFTER:
     - STORAGE_CONNECTION_STRING plaintext in app settings
     - Vendor API private key still in git history (commit 1a8d413)
     - Function subnet delegated to wrong service (Microsoft.App/environments)

   [k]eep  [e]dit  [s]kip  ?
```

If the user picks `[e]dit`, drop into a short back-and-forth to refine that specific field. Then re-show the final version.

## Step 6: Confirm the full batch
After all per-field decisions, show the complete diff one more time as a single atomic update, then ask: "Write to Pulse? [y/N]"

## Step 7: PATCH Pulse
Build the single PATCH payload with only the fields the user confirmed:
```json
{
  "brief": {
    "objective": "...",       // only if changed
    "currentReality": "...",  // only if changed
    "risks": [...]            // only if changed
  },
  "nextActions": [...],       // only if changed
  "history": [...]            // always — append new entry to existing array
}
```

**Critical:** for `history`, fetch the existing array first and APPEND the new entry, don't overwrite. Include `{date, action, by}` where `by` is `<YOUR_EMAIL>`.

Send via:
```
curl -X PATCH "https://<YOUR_DASHBOARD_DOMAIN>/api/projects/<taskId>" \
  -H "Content-Type: application/json" \
  -d '<payload>'
```
(SWA-authenticated path. If that fails, fall back to Playwright-via-browser as in Step 2.)

## Step 8: Report
Confirm each field that was written with a one-line status:
```
✅ Pulse updated (taskId: <id>)
   history     → +1 entry
   nextActions → 4 items (was 3)
   brief.risks → 3 items (was 2)
   brief.currentReality → unchanged
   brief.objective → unchanged
```

If the update partially succeeded (e.g., history wrote but nextActions failed), tell the user exactly which fields made it and which didn't so they can manually reconcile.

## Constraints
- **Never write to Pulse without an explicit `[y]` on the final batch confirmation.**
- **Never overwrite `history`** — always fetch + append.
- **Never touch Planner-managed fields** (title, percentComplete, dueDate, assignees).
- **Never invent owners or due dates** for next-action items. Use `null` if unknown and ask the user to fill them in manually if they care.
- **Never propose changes based on assumptions** — only on work that actually happened in this session. If you don't know what changed, ask before proposing.
- **If the task has no anchor in ~/CLAUDE.md**, you can still update Pulse (the taskId is enough), but flag it: "This project isn't anchored in CLAUDE.md yet — consider running /planner-link to join them."
- **Don't mix this with /planner-push.** If the user wants % complete updated too, suggest running /planner-push as a separate step.
