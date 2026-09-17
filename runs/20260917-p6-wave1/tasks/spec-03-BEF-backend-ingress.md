# SPEC-03: BE-F — scraper/ingestion backend (raw_ingress + adapters + endpoint)

> Run wave2 · EaseApply (/workspace/platform) · builder unit ≤30 min each
> Provenance: RESEARCH-aimode-thread.md §1 (orchestrator 3×/day, admin-only
> scrape) + RESEARCH-infra-orchestration.md. User directives: every backend
> feature gets UI, no Docker, uv policy.

## Context (read only what's named)
- `backend/app/routers/jobs.py` — existing mock `/jobs/scrape` run + seed
  conventions; `backend/app/routers/portals.py` — portal registry
  (`/tools/portals` POST/test), `market` field conventions.
- `backend/app/pg_mixins/jobs.py` — LIVE_JOB_STATUSES ('new'+'active'):
  BE-F ETL sets `status='active'` on real ingestion; the 14-day sweep already
  ages both. `postedAt` ISO strings (naive-UTC ok, normalized in _age_days).
- Upstream reference (port, not copy): `/workspace/ai-job-search/.agents/skills/`
  portal-search CLIs (arbeitnow-search, freehire-search) — HTTP API shape,
  tag filtering, attribution fields (you contributed linkedin/freehire
  attribution upstream: commit 77883a1 lineage).

## Deliverables (files this unit owns — disjoint from SPEC-04/05)
1. `backend/app/services/ingress.py` (new): `raw_ingress` model — store raw
   scraper payloads (source, fetched_at, payload JSON, processed flag) before
   ETL. Dual-mode: MockDB in-memory dict + PG write-through (mirror the
   `_persist_job_fields` literal-SQL pattern; a `raw_ingress` table needs a
   migration — document the compromise, mock-first, `CREATE TABLE IF NOT
   EXISTS` guard acceptable for PG mode).
2. Adapter pattern: `fetch_arbeitnow(http) -> list[dict]` + `fetch_freehire`
   using urllib/requests-free stdlib (no new deps; httpx NOT available in
   backend — check first). Each adapter: timeout, error handling at I/O
   boundary only, returns normalized wire dicts (title, company, location,
   salary?, portal, url, description, skills, postedAt, matchScore=0,
   isRemote, market, status='active'). Attribution: keep `portal` set to the
   board id; no personal-data fields.
3. ETL: `ingest(database, jobs: list[dict]) -> int` — dedupe by (url) against
   existing jobs + within batch; upsert via the jobs bridge (status='active'
   on new ingest); return ingested count.
4. Endpoint `backend/app/routers/portals.py` (or jobs.py if cleaner — planner
   decision, note it): `POST /tools/portals/{id}/ingest` (admin-only,
   require_admin) — runs the adapter for a registered portal, stores raw
   payload, ingests, returns {ingested, deduped, errors}. A `GET /admin/ingress`
   listing recent raw payloads (processed flag) for the panel.
5. Tests `backend/tests/test_ingress.py` (TDD): name cases —
   `test_ingest_dedupes_by_url`, `test_ingest_sets_active_status`,
   `test_adapter_normalizes_and_handles_timeout` (mock http via monkeypatch),
   `test_ingest_endpoint_admin_only`, `test_raw_ingress_stored`.

## QUALITY (gates v4)
SRP (adapters pure-ish, ETL separate, endpoint thin) · ≤3 nesting · arity ≤3
(kwargs) · I/O-boundary error handling · no hardcoded config (timeouts,
URLs from portal registry, 14d window referenced from one place) · no ≥5-line
duplication (shared normalize helper) · fn 50 / file 300 / complexity 10 /
depth 3 · QA: cd /workspace/platform/backend && UV_CACHE_DIR=.uv-cache
UV_LINK_MODE=copy uv run ruff check . && uv run pytest -q (PG tests excluded
per RUN.md — stale TEST_DATABASE_URL).

## Checkpoints
ingress.py → tests → adapters/ETL → endpoint → full QA → "## Result".
Never >1 step unwritten.

## Result
BUILT INLINE by orchestrator 19 Sept 2026 (builders fast-fail died — 10th consecutive engine death).
- services/ingress.py: adapters (arbeitnow/freehire via httpx — IS available
  in backend; urllib fallback), shared normalize shape, raw_ingress store
  (lazy-init MockDB attr; PG CREATE TABLE IF NOT EXISTS — migration = P5-4
  follow-up), ETL ingest (dedupe by URL, propagate to ALL owner spaces per
  the seed-copy convention — u-admin-only landing made jobs invisible, caught
  in build), mark_processed.
- routers/portals.py: POST /portals/{id}/ingest (admin-only; ADAPTERS keyed by
  REGISTRY ids 'arbeitnow'/'freehire' — my spec said '-search' ids, registry
  is source of truth) + GET /admin/ingress (payload stripped).
- tests/test_ingress.py: 8 cases. QA: 59/59 combined; ruff = B008 idioms only.
