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
BUILT INLINE by orchestrator 17 Sept 2026 (after 2 builder fast-fail deaths).
- pg_mixins/jobs.py: LIVE_JOB_STATUSES ('new'+'active'), JOB_STALE_DAYS=14,
  _age_days (naive/aware normalization — mock seeds are naive-UTC),
  expire_stale_jobs (lazy on read), report_job_closed (instant flip +
  _closedReport audit namespaced in gates JSONB — compromise documented,
  proper column = P5-4 migration), reopen_job (cross-owner via find_job_owner —
  SQL SELECT in PG, mock dict scan), _persist_job_fields (literal SQL only,
  S608 gate; mock no-op).
- routers/jobs.py: sweep wired into list_jobs BEFORE the dict copy (bug caught:
  sweep-after-copy made flips invisible); POST /jobs/{id}/report-closed;
  POST /admin/jobs/{id}/reopen (require_admin). Initial admin 404 bug: mixin
  searched the ADMIN's job list — fixed with cross-owner resolution.
- db.py: _posted_days_ago helper (module-level); demo seed postedAt now
  relative (3/5/8/12/22/30/45/50 days ago) — hardcoded March 2026 dates made
  the whole demo board stale under the 14-day sweep (gate exposed it).
  Deadlines still fixed March dates — cosmetic, ledger follow-up.
- frontend: client.ts reportJobClosed (dual-mode) + types JobStatus/Job.status/
  JobGates.closedReport + JobDetail.tsx "Report as Closed / Broken Link"
  (ghost/destructive-subtle, hidden when expired, toast, optimistic setJob).
- Tests: tests/test_jobs_expiry.py 8 cases (isolated via pro-space twin
  fixture with restore). QA: backend 45/45 pass; frontend tsc 0, eslint
  0 errors (19 pre-existing warnings incl. JobDetail max-lines 519>300 —
  ledger), vitest 52/52, prettier clean.
