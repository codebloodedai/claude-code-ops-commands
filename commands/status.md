---
name: status
description: Cross-project status check — pull current state from ~/CLAUDE.md + vault + git
---

Give me a fast cross-project status check. Do the following:

1. Read `~/CLAUDE.md` "Leadership Priorities" section and list each priority with its owner + % complete.
2. For each priority that's owned by You, check if the corresponding vault note in `~/vault/projects/` has been updated in the last 7 days. Flag stale ones.
3. Run `cd ~/Code/<repo> && git status -s` on every repo listed in `~/CLAUDE.md` "Project Repos" table. Report any with uncommitted changes.
4. Run `cd ~/Code/<repo> && git log --oneline origin/main..HEAD 2>/dev/null | head -3` on the same repos. Report any with unpushed commits.
5. Output a single summary table: Priority | Owner | % | Vault freshness | Git state.

This is a read-only recon. Do not modify anything.
