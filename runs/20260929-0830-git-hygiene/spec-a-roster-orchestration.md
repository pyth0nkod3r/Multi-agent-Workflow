# Spec A — Roster + ORCHESTRATION amendments: git-commit hygiene (incident 6ac1e91)

Repo: /workspace/multiagent (canonical roster source + ORCHESTRATION.md). Read the current builder.md + ORCHESTRATION.md §B2 FIRST.

## Incident being prevented (28-29 Sept)
A parallel worker re-applied a fix (commit 6ac1e91, libpq options param) that another session had already REVERTED (cccff8f) — its knowledge was stale, its commit landed on top of the fresher state, and the wrong code deployed to prod (Render rebuilt from the old commit). Also: orphan commit e2ac9b3 appeared with no git run in its turn (provenance untraceable); 7 partial-writeback deaths to date (builders claim done, disk incomplete).

## Amendments to deploy (edit the files, never rewrite wholesale — preserve all existing content)

### builder.md — add a GIT HYGIENE section (near the checkpoint/RESULT-LAST rules)
1. **ORCHESTRATOR-ONLY COMMITS: builders NEVER run `git commit` or `git push`.** Builders write files to disk, fill ## Result, and report. The orchestrator (who holds session context + commit lineage + just ran QA) commits after disk verification. A builder that needs a commit notes "needs commit: <files>" in its report instead.
2. **VERIFY-BEFORE-FIX: never fix from dispatch-time memory.** Before touching any file: re-read it + run `git log -1 --format="%h %s" -- <file>`. If the issue is already fixed/reverted in a newer commit, STOP and report "already fixed in <hash>, skipping" — re-applying a reverted fix is the incident class this rule exists for.
3. **RUN-ID MARKERS: every commit message carries [run <runid>]** (builders note the run id in their report; orchestrator uses it in the commit message).

### ORCHESTRATION.md — add to §B2 (wave quality gating area) a GIT-COMMIT HYGIENE block
1. Orchestrator-only commits (builders never git commit/push) — serialized single writer eliminates commit-ordering races + gives every commit provenance.
2. Orchestrator pre-commit protocol: `git pull --rebase` → re-read every file the commit touches → run the commit with [run <runid>] marker → push → verify remote HEAD moved.
3. Orphan-commit audit: any commit without a run-id marker gets audited (git show + cross-check against active runs) before it's trusted.
4. Incident audit trail: reference the 6ac1e91/cccff8f chain (wrong fix re-applied over fresher state; deployed stale; pooler rejected the options param) as the motivating case.

## Quality gates
Preserve existing content byte-for-byte where untouched (memory id 15: never rewrite paths or drop sections). Functions/simplicity rules don't apply to prose; keep amendments tight (each rule 2-4 lines).

## Checkpoint-or-die
Checkpoint after every step. Fill ## Result LAST after disk verification (git diff shows the amendments present + no existing content lost). Then report. Do NOT commit or push — the orchestrator owns git (per amendment 1 — this spec's own commit is the orchestrator's).

## Result
(Fill after disk verification)
DONE: Updated `builder.md` with the GIT HYGIENE section (rules for orchestrator-only commits, verify-before-fix, and run-id markers) right before the QUALITY FLOOR rule. Updated `ORCHESTRATION.md` by appending the Git-commit hygiene block to section B2. Disk verification confirms all original content remains untouched except for these additions.
