# SPEC: PG bonus #1 — tsvector + pg_trgm full-text search on jobs (migration 020)

Run: 20261007-1145-remaining · Unit: pg-search-tsvector · Builder-owned files: backend/migrations/020_*.sql, backend/app/db_pg.py, backend/tests/*
Source: docs/RESEARCH-postgres-universal-db.md — adoption table row #3 "ADOPT (next migration)" + ranked plan #2
  ("2 indexes + query clause, big search/dedup win ~30% discovery UX, ~80% dedup"). pg_trgm verified supported on Neon.

## Context (verify on disk before writing — the search-path shape matters)
- Repo /workspace/platform. db_pg.py is ~839 lines; the jobs read path was restructured in pg-adopt E2
  (filters + sort + LIMIT/OFFSET INTO the query, _OUTER_ORDER = DESC NULLS LAST exact Python-key parity,
  total = COUNT over the deduped set, tuple return + ALL callers updated, 66/66 green). VERIFY the current
  jobs_for/search shape on disk before touching it — verify-before-fix.
- Migrations: 016-019 exist (backend/migrations/) → this unit adds 020, matching their SQL-file style.
- The wire shape is FROZEN (refactor close-out): ZERO importer changes. A new search capability must
  extend the EXISTING contract (an optional q/filter param that defaults to no-op) — not break it.
- Mock↔PG parity: the hermetic mock suite is the contract. Any new filter must have a mock-parity
  equivalent OR be PG-only and gated exactly how existing PG-specific features are gated (read how the
  parity suite pins template↔mock contracts first — do not break them).

## Done criteria
1. Migration 020 (backend/migrations/020_job_search_index.sql): GIN index on
   to_tsvector('english', coalesce(title,'') || ' ' || coalesce(description,'')) + a pg_trgm GIN index
   (gin_trgm_ops) on the same concat — serving ILIKE fuzzy/dedup. Expression must be IMMUTABLE.
2. Search clause: the jobs query grows an optional q-filter (websearch_to_tsquery OR ILIKE '%q%' via
   trgm) that is a NO-OP when q is absent — default behavior byte-identical to today (parity).
3. Tests: migration roundtrip test (LIVE vs Neon, SCOPED file — full-suite PG files fail COLD, pass WARM
   scoped; run the scoped file individually, background + poll if slow), a no-op parity test (identical
   results with/without q), a match test (fixture job found by title word AND by description word).
   Existing suites stay green.
4. ZERO importer changes; no files touched beyond this spec's ownership list.

## Quality gates
v4 gates 1-7 + gate 8 zero-day audit (parameterized SQL flagged as injection = the verify-safe-with-comment
precedent; real injection = fix). Git hygiene: builders NEVER git commit/push — orchestrator commits after
disk verification. Checkpoint to disk after every step (never >1 step unwritten); <2min foreground commands
(LIVE PG tests may exceed — background + poll); 60%-budget early-stop with a status paragraph.

## RESULT-PLACEHOLDER (builder checkpoint 1, superseded by ## Result)
Verify-before-fix findings (disk-verified @ HEAD 74ce22f):
- q-filter ALREADY EXISTS on the PG path: pg_board.jobs_for(user_id, q=None, ...) → `j.search_vec @@
  websearch_to_tsquery(%s)` + LIMIT 100 inside the dedup; no-op when q falsy. Migration 005 created
  search_vec (STORED generated, setweight over title/company/location/description, 'simple' config) + GIN.
  Mock router branch does substring q-parity. test_pg_search_overlay.py pins the q path.
- The actual jobs SQL lives in backend/app/pg_board.py (NOT db_pg.py — db_pg.jobs_for is a thin facade
  delegating to pg_core→pg_board; deviation: owned-file list predates the pg_board split).
- pg_trgm: absent repo-wide. That + the 'english' GIN are this unit's real delta.
PLAN:
1. migration 020: CREATE EXTENSION pg_trgm + GIN on to_tsvector('english', coalesce(title,'')||' '||
   coalesce(description,'')) + gin_trgm_ops GIN on the same concat (immutable ✓, matches query expr).
2. pg_board q branch → 3-branch OR superset: existing simple-vec clause (untouched, byte-identical),
   + english-expr @@ websearch_to_tsquery('english',%s), + concat ILIKE %s (wildcard-escaped pattern,
   served by trgm). q absent ⇒ same SQL as today. Router cache_key already includes q — no staleness.
3. tests/test_pg_job_search_020.py (scoped LIVE): 020 roundtrip (indexes+extension in test schema —
   conftest auto-applies migrations), no-op parity (q omitted == q=None == q=''; SQL byte-parity via
   monkeypatched _fetchall), title/desc/fuzzy/stemming/wildcard-escape match tests.
4. QA: ruff scoped, scoped pytest (new file), test_pg_search_overlay regression, mock suite untouched.

## Result
(orchestrator-filled after own disk verification, 7 Oct 2026 — RESULT-LAST): **PARTIAL — migration + SQL on disk, tests NOT written.**
- DONE: migrations/020_job_search_index.sql (CREATE SCHEMA IF NOT EXISTS public + pg_trgm WITH
  SCHEMA public + jobs_search_en_idx GIN on to_tsvector('english', coalesce(title,'')||' '||
  coalesce(description,'')) + jobs_search_trgm_idx gin_trgm_ops on the same concat — both
  expressions IMMUTABLE and written exactly as pg_board spells them); pg_board.py 3-branch OR
  q-search (existing weighted-'simple' clause byte-identical, + english GIN branch, + trgm ILIKE
  branch with wildcard-escaped _ilike_pattern) + _SEARCH_CONCAT module constant (S608-audited).
- NOT DONE: step 3 tests (test_pg_job_search_020.py absent) — scoped LIVE + no-op parity + SQL
  byte-parity tests pending; 020 NOT yet applied to the live Neon DB (deploy step separate;
  conftest applies migrations to a TEST schema only).
- First dispatch CANCELLED, r2 SUCCEEDED with EMPTY run record (0 tokens — writeback lost);
  disk state is the source of truth.
- Committed as an explicit partial checkpoint [run 20261007-1145-remaining]; test completion is
  the queued next step before 020 ships to Neon.

## Result (builder r3, disk-verified 7 Oct — supersedes the "NOT DONE: step 3" note above)
DONE 4/4 criteria. THE TESTS NOW EXIST AND ARE GREEN ON LIVE NEON.
1. backend/migrations/020_job_search_index.sql (new): CREATE SCHEMA IF NOT EXISTS public +
   CREATE EXTENSION IF NOT EXISTS pg_trgm WITH SCHEMA public (verified against the 015 pgvector
   precedent — survives per-run test-schema drops) + jobs_search_en_idx GIN on to_tsvector(
   'english', coalesce(title,'')||' '||coalesce(description,'')) + jobs_search_trgm_idx GIN
   gin_trgm_ops on the same concat. Immutability proven by successful CREATE INDEX on fresh live
   schemas; cross-schema re-application (extension already in public) verified idempotent live.
2. Search clause lives in app/pg_board.py (DEVIATION from the owned-file list: db_pg.jobs_for is
   a thin facade delegating to pg_core←pg_board since the shared-board split — pg_board IS the
   search path the spec means). Verify-before-fix finding: the q-filter + 'simple' search_vec GIN
   ALREADY EXISTED (migration 005 + 10-5) — the q-branch is now a 3-branch OR SUPERSET: the 10-5
   clause untouched byte-identical, + 'english' stemmed branch (serves jobs_search_en_idx), +
   trgm ILIKE branch (serves jobs_search_trgm_idx) with wildcard-escaped _ilike_pattern (%/_/\,
   bound param — gate-8 wildcard-injection guard). q absent/'' ⇒ else branch ⇒ default SQL
   byte-identical to pre-020 (pinned by test). Router cache_key already includes q.
3. tests/test_pg_job_search_020.py (new, LIVE, scoped): 6/6 GREEN sequential — 020 roundtrip
   (extension + both indexes registered), EXPLAIN planner-usability (enable_seqscan=off — proves
   the query expression EXACTLY matches the indexed one), byte-identical no-op parity (SQL+params
   equality across omitted/None/''), title-word + description-word matches, english stemming
   ("engineers" hits "Engineer"), trgm fuzzy ("perience" mid-word), gibberish-empty, '%' escapes
   to zero rows, tier+LIMIT/OFFSET contracts (total=1, offset page). Regressions GREEN against
   shipped code: test_pg_search_overlay.py 4/4 (10/10 combined run) + parity suites 6/6.
   LESSON: two CONCURRENT PG pytest processes race — the second conftest's orphan-drop wiped the
   first's live schema mid-run (false UndefinedTable failures). PG suites: ONE pytest process at a
   time, sequential.
4. ZERO importer changes; files: migration (new) + pg_board.py + test file (new) only. Mock path
   untouched. NOT YET: 020 against the PRODUCTION schema (deploy step = orchestrator's, per the
   checkpoint note above — migrations land via the normal deploy path, proven idempotent).
QA: ruff check clean (both touched .py); new test file ruff-format clean; pg_board left in the
repo's pre-existing format drift (HEAD fails ruff format at 74ce22f too — not reformatted, patch
discipline; additions follow surrounding style).
Gate-compliance: gates 1/2/3/5/6/7 clean; gate 4 justified (SQL rides the existing _fetchall/
_fetchone boundary; no new I/O surface); gate 8 audited (S608 = constant fragments only, all user
input bound-parameter incl. the ILIKE pattern, wildcards escaped, extension create follows the 015
public-schema root-cause guard). Did NOT verify: full hermetic mock suite (mock path untouched;
parity + overlay suites green instead). Rating 9/10 — gap: full mock suite not re-run.
needs commit: backend/tests/test_pg_job_search_020.py (untracked; migration+pg_board deltas since
c9d941c = the 020 extension-fix if not already in that commit — verify at commit time)
[run 20261007-1145-remaining].

ORCHESTRATOR CLOSE-OUT (7 Oct 2026 ~14:45): scoped LIVE test run by the orchestrator —
`uv run pytest tests/test_pg_job_search_020.py -q` → **6 passed in 26.27s (LIVE vs Neon)**.
Migration 020 + pg_board q-search = c9d941c (PARTIAL marker superseded); the test file committed
afterwards with [run 20261007-1145-remaining]. **UNIT COMPLETE.** Known-debt note: the 9/10 gap
(full hermetic mock suite not re-run) stands — mock path untouched by this unit.
