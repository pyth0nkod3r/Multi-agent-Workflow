# Task 01 — Builder: make test_pg_drafts.py pass on Neon, commit §5

## Repo
`/workspace/platform` (git@github.com:pyth0nkod3r/job-application-suite.git, branch main).
Work in `/workspace/platform/backend`. There are UNCOMMITTED changes (drafts mixin §5) —
do NOT stash/discard them; they are the feature under test.

## Context (verified by orchestrator 21:40 UTC)
- `tests/test_pg_drafts.py` — 5 tests, currently HANGS (>4 min) under pytest against
  Neon Postgres (`TEST_DATABASE_URL` in `backend/.env`).
- Schema `jobplatform_s5` on Neon is migrated (all tables exist, migrations 1–3 recorded).
- The pg test pattern to copy: `tests/test_pg_fit.py` (PG_SCHEMA=jobplatform_s4,
  dependency override get_db → PgDB, TRUNCATE+reseed per test) — it passes.
- MockDB parity reference: `app/db.py` (templates as dicts, drafter/reviewer/compile
  endpoints in `app/routers/tools.py`).
- The new code under test: `app/pg_mixins/drafts.py` (+123 lines, `templates` /
  `custom_templates` registries with WriteThroughList items), wiring in
  `app/db_pg.py`, `app/pg_core.py`, `app/routers/tools.py`.

## Steps
1. Reproduce: `cd /workspace/platform/backend &&
   export $(grep -m1 '^TEST_DATABASE_URL' .env) && PG_SCHEMA=jobplatform_s5
   uv run pytest tests/test_pg_drafts.py -v` (run in background, poll; kill after ~3 min).
   Identify WHICH test hangs and why — inspect what that endpoint path calls
   (`app/routers/tools.py` → drafter/reviewer/compile) and what the mixin / deps
   do on first call. Suspects: a network call not mocked, a lock, a lazy seed
   that blocks, or an infinite loop in the WriteThrough wrappers.
2. Diagnose with the smallest possible probe (single test, `-x`, prints, direct
   function calls outside pytest). Keep each command <3 min — use timeout.
3. Fix minimally. The bug is likely in the new uncommitted code or its wiring —
   NOT in the passing s4 pattern. Do not re-architect. If the fix requires
   touching `test_pg_drafts.py` (e.g. a missing fixture), keep it in the same
   style as test_pg_fit.py.
4. Verify: all 5 tests green in a single run
   (`PG_SCHEMA=jobplatform_s5 uv run pytest tests/test_pg_drafts.py -q`), AND
   the neighbour suites still pass: `PG_SCHEMA=jobplatform_s4 uv run pytest
   tests/test_pg_fit.py tests/test_pg_tracker.py -q`.
   Also run the hermetic mock suite: `uv run pytest tests/ -q -k "not pg_"`.
5. Commit: stage ONLY the §5 files
   (`app/pg_mixins/drafts.py app/pg_core.py app/db_pg.py app/routers/tools.py
   tests/test_pg_drafts.py` + any fix-related files; never .env),
   message `feat(backend): WS-B2 §5 — drafter-reviewer pipeline + template registries on Postgres`.
   Do NOT push; the orchestrator pushes after critique.

## Output
REPLACE the `## Result` placeholder below with:
- root cause (1–3 sentences),
- the fix (files + what changed),
- verification: exact commands run + pass/fail counts for all three suites,
- commit hash + `git show --stat HEAD` output,
- anything you could not verify.

## Result (builder-2 + orchestrator, complete)
- Root cause of the original "hang": Neon latency — all 5 tests pass in ~81s.
  No deadlock. No product code fix needed for §5; the schema issue (`global_portals`
  missing in jobplatform_s5) was an unrecorded migration, fixed by the orchestrator
  pre-dispatch (INSERT schema_migrations id=1, then all 3 recorded).
- Fix: none required in the drafts mixin. Uncommitted §5 work (drafts mixin +
  pg_core/db_pg/router wiring + test file) verified as-is and committed.
- Verification:
  - `PG_SCHEMA=jobplatform_s5 uv run pytest tests/test_pg_drafts.py -q` → 5 passed (~81s)
  - `PG_SCHEMA=jobplatform_s4 uv run pytest tests/test_pg_fit.py tests/test_pg_tracker.py -q` → passed (builder-2)
  - Hermetic mock suite, pg files excluded:
    `uv run pytest tests/ -q --ignore=tests/test_pg_drafts.py --ignore=tests/test_pg_fit.py --ignore=tests/test_pg_jobs.py --ignore=tests/test_pg_tracker.py --ignore=tests/test_db_pg.py` → 85 passed in 9.94s
  - NOTE: the spec's original step-4 command (`-k "not pg_"`) is invalid for this
    codebase — pg test files still get COLLECTED, and their module-level seed_pg
    hits the process-global cached connection whose search_path was pinned by
    whichever module imported first → spurious UndefinedTable. Orchestrator
    decision: pg suites always run in their own process (as (a)/(b) do); mock
    suite excludes pg files via --ignore.
- Commit: 4476c37 `feat(backend): WS-B2 §5 — drafter-reviewer pipeline + template registries on Postgres`
  ```
  backend/app/db_pg.py            |   2 -
  backend/app/pg_core.py          |   5 --
  backend/app/pg_mixins/drafts.py | 123 +++++++++++++++++++++++++++++++++++++++-
  backend/app/routers/tools.py    |   5 ++
  backend/tests/test_pg_drafts.py | 114 +++++++++++
  5 files changed, 239 insertions(+), 10 deletions(-)
  ```
- Not done: push (orchestrator holds until critic PASS); no critic verdict yet.