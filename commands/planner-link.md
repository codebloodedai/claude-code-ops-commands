---
name: planner-link
description: One-time bootstrap — match Planner tasks to ~/CLAUDE.md strategy bullets and embed stable anchors
---

Run this ONCE (or when adding new tasks) to bootstrap the Planner ↔ CLAUDE.md join keys. After this runs, /planner-pull and /planner-push can find every bullet deterministically.

## Step 1: Get your Planner user ID
```
az rest --method GET --url "https://graph.microsoft.com/v1.0/me?\$select=id,displayName"
```

## Step 2: Pull tasks from all 3 active plans
Same as /planner-pull step 2. For each task, extract: `id`, `title`, `percentComplete`, `planId`.

Plan IDs (authoritative list — source of truth is `~/CLAUDE.md` "Microsoft Planner Reference"):
- **IT Operations**: `<YOUR_PLAN_ID_1>`
- **Enterprise Engineering & Operations**: `<YOUR_PLAN_ID_2>`
- **AI Initiatives**: `<YOUR_PLAN_ID_3>`

Filter to tasks assigned to you OR with no assignee (orphans should be flagged).

## Step 3: Read ~/CLAUDE.md and extract existing bullets
Read `~/CLAUDE.md`. For every line matching:
```
- [ ] <text> `(<owner>)` `<pct>%`  [optional trailing — comment]
- [x] <text> `(<owner>)` ...
```
capture: line number, checkbox state, full text, owner, pct, whether it already has a `<!-- planner:... -->` anchor.

## Step 4: Match bullets to tasks
For each strategy bullet without an anchor, score every Planner task by text similarity against the bullet's text. Use a simple approach:
- Normalize both to lowercase, strip punctuation, drop stopwords (the, a, an, to, of, for, in, on, and)
- Compute token overlap / Jaccard similarity
- Prefer tasks in the same "domain" — bullets under OBJ-1 sections are more likely to match IT Operations plan tasks, OBJ-4 bullets match Enterprise Engineering

For each bullet, keep the top 3 candidate matches with their similarity scores.

## Step 5: Show the proposed mapping
Display a table like:
```
BULLET                                              → CANDIDATE MATCHES
(L45) Ship monitoring integration (you) 10%  → 1. "Monitoring integration" (95%) [IT Ops]  <EXAMPLE_TASK_ID>
                                                    → 2. "SIEM Implementation" (42%)                    [IT Ops]  bB-67cfwJE-VYOw...
                                                    → 3. (no other candidates above 30%)

(L52) Harden password manager posture (you) 75%    → 1. "improve password security posture" (88%) [IT Ops]  <EXAMPLE_TASK_ID_2>
                                                    → 2. "Integrate password manager with Azure SSO" (40%)        [IT Ops]  2AusZtjQ-UaGdW...
```

Also show:
- **Unmatched bullets** (no candidate above 30% similarity) — these need manual linking or are not in Planner
- **Unmatched Planner tasks** (no bullet matched) — these are in Planner but missing from CLAUDE.md, might need adding

## Step 6: Interactive confirmation
For each bullet-to-task mapping, ask: "Accept match? [y/n/s=skip/m=manual taskId]". Default to the top candidate.

Collect all accepted mappings into a batch before writing.

## Step 7: Write the anchors
For each accepted mapping, use the Edit tool to append ` <!-- planner:<taskId> -->` to the bullet line. Keep the existing bullet text + owner + pct unchanged.

Example:
```
old_string: - [ ] Ship monitoring integration `(you)` `10%`
new_string: - [ ] Ship monitoring integration `(you)` `10%` <!-- planner:<EXAMPLE_TASK_ID> -->
```

Make ONE Edit tool call per bullet. Don't batch them into a single multi-line replacement — that's fragile.

## Step 8: Report
Show:
- N bullets received new anchors
- N bullets skipped
- N Planner tasks still unmatched (candidates for new bullets)
- N CLAUDE.md bullets still unmatched (not in Planner)

Suggest creating any new strategies for the orphans the user wants to track.

## Constraints
- **Never write an anchor without explicit confirmation** for that specific bullet. Bulk "accept all" is OK only if the user explicitly asks for it.
- Never modify a bullet that already has an anchor — those are already joined.
- Never change the text/owner/pct of a bullet during this operation. This is anchor-only.
- If a task similarity is below 30%, don't propose it as a match at all.
