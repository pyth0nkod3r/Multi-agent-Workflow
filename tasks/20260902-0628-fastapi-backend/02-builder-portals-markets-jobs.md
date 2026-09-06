# 02 — Builder: Portals + Markets + Jobs/scrape + Dashboard routers

Contract: `tasks/20260902-0628-fastapi-backend/00-CONTRACT.md` (read first).

## Deliverable
`backend/app/routers/portals.py`, `backend/app/routers/markets.py`, `backend/app/routers/jobs.py`, `backend/app/routers/dashboard.py` — plus tests in `backend/tests/test_jobs_portals_markets.py`.

## Endpoints

### portals.py — tag [Portals]
- `GET /portals` → 200 list. Merge: global portals + registered_portals; then apply per-user overrides (db.portal_overrides[user_id] is {portal_id: enabled}); return copies (never mutate the globals).
- `PATCH /portals/{portalId}` — require_admin. Body PortalToggleRequest. Store override `{portalId: enabled}` for the ADMIN's own user scope (mock simplification: admin toggles are global-acting but stored per-user). 404 if portal id unknown (not in globals or registered).

### markets.py — tag [Markets]
- `GET /markets` → 200 all markets with `active` reflecting db.user_markets[user_id] (default seed: ["de", "ie"]).
- `GET /markets/active` → 200 only active ones.
- `POST /markets/{code}/activate` — require_admin. Add code to user_markets if not present; return full list with active flags. 404 unknown code.
- `POST /markets/{code}/deactivate` — require_admin. Remove code. 404 unknown.

### jobs.py — tag [Jobs]
- `GET /jobs` → 200 list (db.jobs[user_id]) sorted by matchScore DESC. Query filters (all optional, combinable): `portal` (exact), `minScore` (>=), `market` (exact), `fitVerdict` (exact).
- `POST /jobs/scrape` — require_admin. Body ScrapeConfigPatch (optional; treat absent body as {}). Behavior: mark run summary from the user's enabled portals; jobsFound = random-ish deterministic 10-30 (use len(portals)*4+7 — deterministic, testable); newJobs = jobsFound - 3 (min 0); skippedDisabled = requested portals that are disabled; fallbackWebsearch = ["jobs-ie"] if included and source=="websearch"; health = one entry per enabled portal (status "healthy", detail f"HTTP 200, N jobs (mock)"). Store run in db.scrape_runs[user_id]; ALSO append 2 new mock jobs to db.jobs[user_id] (ids "s1"/"s2", portal from config, matchScore 70/55, fitVerdict good_fit/moderate_fit, status "new", today's postedAt) so a scrape visibly grows the list. Update scrape config if body provided. Return ScrapeRunSummary shape.
- `GET /jobs/scrape-config` → 200 config.
- `PATCH /jobs/scrape-config` → 200 updated config (apply only provided fields).
- `GET /jobs/portal-health` — require_admin. → 200 one PortalHealth per enabled portal: healthy with deterministic detail; the portal with id "jobs-ie" gets status "inconclusive", detail "DuckDuckGo HTML rate-limited (0 results) — not evidence of breakage".

### dashboard.py — tag [Dashboard]
- `GET /dashboard/stats` → 200 `db.dashboard_stats(user_id)`.

## Test file (backend/tests/test_jobs_portals_markets.py)
Cover (use conftest fixtures user_headers/admin_headers — they exist):
- /jobs without auth 401; with user_headers 200, sorted desc by matchScore, 8 items
- /jobs?minScore=85 returns only >=85; ?portal=arbeitnow filters; ?market=de filters; ?fitVerdict=poor_fit returns ALDB
- POST /jobs/scrape with user_headers → 403 (admin-only!); with admin_headers → 200 summary has jobsFound/newJobs/skippedDisabled/health fields; db grew by 2 jobs
- PATCH /jobs/scrape-config updates maxResults; GET reflects it
- /jobs/portal-health: admin 200 with jobs-ie "inconclusive"; user 403
- /portals: user 200 list of 9; PATCH /portals/arbeitnow as user → 403, as admin → 200 enabled flipped
- /markets: user GET 200 (de+ie active); activate as user → 403; as admin activate "uk" → 200 uk active; deactivate "de" → 200; unknown code "xx" → 404
- /dashboard/stats: user 200 with totalScraped == len(jobs), topSkills/sourceBreakdown lists

## Done criteria
- `uv run pytest tests/test_jobs_portals_markets.py -q` green
- `uv run python -c "from app.main import app"` OK

## Result
**Status**: DONE (orchestrator-verified — narration truncated, artifacts confirmed).

**Produced**: routers portals.py / markets.py / jobs.py (rewritten cleanly) / dashboard.py (was missing in r1) + tests/test_jobs_portals_markets.py. Salvaged portals/markets from r1 as the notice permitted; rewritten jobs.py compiles and matches spec shapes (verdict field in run-summary health lines, status reserved for the probe — correct per types.ts).

**Verification (orchestrator)**: all four compile; merged suite 59/59 with the real app.main (conftest fallback no longer active — all 14 routers import).

**rating**: 9/10 (Result block unfilled — filled here from verification)

## RE-DISPATCH NOTICE (orchestrator)
First attempt: jobs.py corrupted (line 174: `user: dict POST /jobs/scrape-config placeholder,` — invalid syntax), dashboard.py missing, no test file, Result block unfilled. This is attempt 2. REWRITE jobs.py cleanly from scratch. The other three routers (portals.py, markets.py) may be salvaged if they compile — verify with py_compile first. dashboard.py and tests/test_jobs_portals_markets.py are still owed.
