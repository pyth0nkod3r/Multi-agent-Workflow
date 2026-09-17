# SPEC-02: BE-E — stale-job lifecycle (14-day expiry + report-closed)

> Run 20260917-p6-wave1 · EaseApply (/workspace/platform) · builder unit ≤30 min
> Provenance: AI Mode thread final turn (RESEARCH-aimode-thread.md §2.1).
> DECISION LOCKED (user-approved default, reversible): 14-day universal expiry;
> community flags flip status INSTANTLY (no moderation queue) + admin-reversible
> trail.

## Context (read only what's named)
- Schema (dual-mode MockDB + PG parity): `backend/app/pg_core.py` `_JOB_COLS` —
  jobs have `posted_at` (expiry key, NOT created_at) and `status` (default
  `'new'`; scrape/ingest sets `'active'` on ingest — verify actual values in
  `pg_mixins/jobs.py` + `backend/app/db.py` seed before coding; if 'active' is
  never used today, the expiry predicate must target the real live-state value).
- `backend/app/routers/jobs.py` — existing endpoint conventions, `_now_iso`,
  MockDB/PG dual-path pattern. Follow it exactly.
- Frontend: `frontend/src/pages/JobDetail.tsx` (478 lines), api lib under
  `frontend/src/lib/api/`. No hard-coded hex (DESIGN.md tokens; shadcn).

## Deliverables (files this unit owns — disjoint from SPEC-01)
1. Expiry sweep `backend/app/pg_mixins/jobs.py` (new function):
   `expire_stale_jobs(database, *, days: int = 14) -> int` — UPDATE jobs SET
   status='expired' WHERE status='<live-value>' AND posted_at < now() -
   interval. MockDB path must mirror PG path (parity by construction).
   Hook: call lazily from `list_jobs` router (cheap, runs on read) — no new
   infra, no cron dependency in this unit; a scheduled orchestrator hook is
   BE-F scope.
2. Report-closed endpoint `backend/app/routers/jobs.py`:
   `POST /jobs/{job_id}/report-closed` — flips status → `'expired'`
   instantly; audit trail `closed_report: {by, at}` stored on the job record
   per existing JSON-column conventions (check how score_breakdown/gates are
   stored); idempotent (already-expired → 200 no-op); 404 unknown id.
   Admin reversal: `POST /admin/jobs/{job_id}/reopen` following existing admin
   auth dependency pattern in routers (check applications.py/admin wiring).
3. Frontend `frontend/src/pages/JobDetail.tsx`: "Report as Closed / Broken
   Link" destructive-subtle action (shadcn button, existing tokens), calls new
   endpoint, optimistic status update + toast; hidden when already expired.
4. Tests:
   - backend `tests/test_jobs_expiry.py`: `test_stale_job_expires_after_14_days`,
     `test_recent_job_not_expired`, `test_report_closed_flips_status`,
     `test_report_closed_idempotent`, `test_reopen_restores_status`.
   - frontend: follow existing src/test conventions (at minimum a render test
     asserting the report action appears for active jobs).

## QUALITY (gates v4)
SRP (expiry logic in mixin, endpoint thin) · ≤3 nesting · arity ≤3 (kwargs
with defaults) · I/O-boundary error handling · no hardcoded config (14-day
window as named constant/setting) · no ≥5-line duplication (reuse existing
job-lookup helpers) · fn 50 / file 300 / complexity 10 / depth 3 ·
`npm run qa` frontend green; backend ruff+pytest green (env vars per RUN.md).

## Checkpoints
Mixin fn + tests → router + tests → frontend → full QA → "## Result".
Never >1 step unwritten.

## Result
(built by — orchestrator fills after disk verification)
