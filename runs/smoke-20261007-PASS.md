# runs/smoke-20261007 — SMOKE PASS (Hermes harness port)

Run: smoke-20261007 | Protocol: ORCHESTRATION §E, Hermes-adapted | Date: 2026-10-07
Harness: Hermes Agent (desktop, Windows). Dispatch channel: `delegate_task`
(RikkaHub `subagent_dispatch` equivalent); role prompts injected from
roster/builder.md v4 (condensed) into each child's context.

## What ran
1. **Scaffold-first**: `tasks/smoke-20261007/` (RUN.md + 3 self-contained specs)
   written to disk BEFORE any dispatch. RUN.md used as write-ahead log.
2. **Wave 1 — fan-out**: 3 builders in ONE parallel dispatch
   (sa-0-ce66a052, sa-1-f0012ccc, sa-2-e8977a8d), results delivered between
   turns (background handoff replaces the watchdog `schedule_job` — no cron
   needed on this harness).
3. **Collect — disk verification (SUCCEEDED ≠ done)**: orchestrator recomputed
   independently: 17×23=391, 2¹²−1=4095, Σprimes<30=129. All three out-*.md
   files on disk with exact correct content; all `## Result` placeholders
   replaced; zero unfilled placeholders.
4. **Wave 2 — blind critic** (sa-0-b0a3c4c9): given only spec+output paths,
   no builder identities. Verdict **PASS**; full report + verification table
   landed at `tasks/smoke-20261007/CRITIC.md` (3543 B, read back from disk per
   critic v5 disk-report rule). Issues: 2 × note, 0 blockers.
5. **Merge**: this file. Critic Issue 1 (H1 copy-paste headers in specs 02/03
   — orchestrator-side scaffold bug) fixed before the commit; correction
   logged in RUN.md (pre-commit fix, no history rewrite).

## Verdict
**PASS** — 3/3 units correct + disk-verified, blind critic PASS with no
blockers, RESULT-LAST and checkpoint discipline observed by all workers.

## Harness translation lessons (ported protocol → Hermes)
- `delegate_task` background handoff = watchdog replacement: results re-enter
  the conversation as messages; orchestrator ends the turn after dispatch,
  never polls transcripts.
- Children are context-isolated and cannot ask the user questions — "return
  blocked in ## Result" must be stated in every role context block.
- RikkaHub's "system_prompt ignored" quirk does NOT apply here: roster prompt
  bodies go into the child's `context` field and work.
- Watchdog `schedule_job` (§A step 4) is optional on Hermes; next-message
  recovery scan remains mandatory and fired once during this run (correctly
  found the in-flight critic, took no action).

## Artifacts
- tasks/smoke-20261007/: 0{1,2,3}-builder.md (specs + filled ## Result),
  out-0{1,2,3}.md, CRITIC.md, RUN.md (full audit trail)
- Commit: `[run smoke-20261007]` — this run + knowledge/README.md index fix
