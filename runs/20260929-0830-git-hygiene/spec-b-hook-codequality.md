# Spec B — Pre-commit run-id hook + code-quality.md gate 10 (incident 6ac1e91)

Repos: /workspace/platform (EaseApply repo — hook + docs) and /workspace/multiagent (knowledge file). Read the current files first.

## Incident being prevented (28-29 Sept)
Orphan commits with no provenance (e2ac9b3 appeared with no git run in its turn); a stale worker commit (6ac1e91) deployed to prod. Mechanical enforcement: a commit-message convention the hook checks, so provenance doesn't depend on the model remembering.

## Files YOU own
- /workspace/platform/.githooks/commit-msg (NEW — the hook)
- /workspace/platform/docs/quality-ledger.md (ONLY: append a ledger note documenting the hook + convention)
- /workspace/multiagent/knowledge/code-quality.md (ONLY: add gate 10)

## Task 1 — commit-msg hook
A POSIX sh hook at /workspace/platform/.githooks/commit-msg:
- Reads the commit message file ($1) first line.
- REJECT (exit 1, message to stderr) commits whose first line does NOT contain a run-id marker matching `[run <id>]` — regex: \[run [0-9a-zA-Z-]+\]. Case-sensitive.
- ALLOW merge commits (first line starts with "Merge ") and revert commits (starts with "Revert ") — but a Revert must still carry the marker if present; only Merge is exempt.
- Keep it ≤20 lines, no dependencies (pure sh + grep).
- Do NOT wire it into .git/config core.hooksPath (that's a local-machine config change — document the wiring command in the ledger note instead: `git config core.hooksPath .githooks` run per-clone).

## Task 2 — code-quality.md gate 10
Append gate 10 to the existing gates (file currently has gates 1-9 — read it first, preserve everything):
**Gate 10 — Git-commit provenance:** every commit message carries a [run <runid>] marker; builders never commit (orchestrator-only commits, serialized single writer); verify-before-fix (re-read target + git log before fixing — never re-apply a reverted fix); orphan commits (no marker) audited before trusted. Enforcement: .githooks/commit-msg (wire per-clone via git config core.hooksPath .githooks).

## Task 3 — ledger note
Append to docs/quality-ledger.md: the hook + convention + the wiring command + the incident reference (6ac1e91 stale-commit deploy → pooler rejected options param → prod down 12h; prevention = orchestrator-only commits + run-id markers + this hook).

## Quality gates
Preserve existing content (no rewrites of untouched sections). Hook ≤20 lines. Docs amendments tight.

## Checkpoint-or-die
Checkpoint after every step. Fill ## Result LAST after disk verification. Then report. Do NOT commit or push — orchestrator owns git.

## Result
- Hook created at `/workspace/platform/.githooks/commit-msg` (POSIX sh, exit 1 on missing `[run <id>]`, exempts `Merge ` commits).
- Quality ledger updated in `/workspace/platform/docs/quality-ledger.md` with incident 6ac1e91 details and the wiring command.
- Gate 10 (Git-commit provenance) added to `/workspace/multiagent/knowledge/code-quality.md`.
