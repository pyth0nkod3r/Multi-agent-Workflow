# RUN.md — 20260907-0405-deploy-prep

Goal: deploy-readiness — backend Render+Neon plumbing; frontend env-gated live-API layer (transport + auth + 3 reads), mock fallback intact.

## Units
- 01-backend: Render blueprint, /health, CORS-from-env, .env.example, DEPLOY.md
- 02-frontend: transport.ts, auth persistence, getJobs/getPortals/getMarkets behind VITE_API_BASE

## Ledger
- 04:10 · scaffold (RUN.md + 01 + 02 + 03) · orchestrator · done
- (dispatches appended below)- 04:25 · 01-backend · f777147c · dispatched background
- 04:25 · 02-frontend · 1d87925e · dispatched background
- 04:40 · attempt-1 both builders · DEAD pre-writeback (frontend notified SUCCEEDED with zero disk output — false success; backend silent, no active runs) — clean tree confirmed
- 04:55 · v2 specs written (inline context, incremental Result writes, no discovery reading) → re-dispatch attempt-2
- 05:15 · attempt-2 both builders · DEAD (backend: import os only; frontend: nothing) — SUCCEEDED status unreliable, 4th consecutive pre-writeback death
- 05:20 · ESCALATION.md written; orchestrator switches to inline execution of specs 01+02; critic still blind-dispatched at end
- 05:20-05:45 · specs 01+02 executed INLINE by orchestrator (4 builder deaths per ESCALATION.md)
- 05:45 · verification (orchestrator): app import ok; uvicorn :8971 /health → 200 {"status":"ok","version":"0.1.0"}; hermetic mock suite 85 passed; tsc --noEmit clean; vitest 1/1; vite build ✓ 1m18s (chunk-size warning only)
- 05:46 · endpoints wired: POST /v1/auth/login, POST /v1/auth/logout, GET /v1/jobs, GET /v1/portals, GET /v1/markets (all flag-gated on VITE_API_BASE; mock fallback unchanged)
- 05:46 · committing: (a) backend deploy plumbing, (b) frontend live-API transport; then blind critic on the diffs
- 05:50 · critic 6d343e4c · PASS, no blockers (dead-code warn: mockLoginUser)
- 06:00 · critic's warn escalated into a REAL bug found by orchestrator: mockProfile referenced at client.ts:98/116 but undefined — earlier tsc "clean" was a pipe artifact (EXIT=$? measured head, not tsc); vite build doesn't typecheck free vars → would've been runtime ReferenceError in mock mode
- 06:01 · fix: mock branches → mockLoginUser; tsc re-run WITHOUT pipe → genuine TSC_EXIT=0
- 06:41 · user's manual Blueprint deploy created srv-daf5p38n74is738f7f10 (free, frankfurt) — my API create hit name-conflict 409
- 06:45 · build_failed diagnosed via /v1/logs: uv.lock gitignored → --frozen found no lockfile. Fix: force-tracked backend/uv.lock (34631f0), removed ignore line
- 06:50 · PATCH service: startCommand = migrate && uvicorn (free tier skips pre-deploy commands); env-vars PUT: CORS_ORIGINS fixed, user's DATABASE_URL preserved verbatim
- 06:55 · deploy dep-daf5rdid0e5s73auqg6g → LIVE on ebdbac7; user confirmed /health 200 + "migrations up to date" in logs
- 06:56 · LIVE ROUND-TRIP VERIFIED from workspace: /health 200 (0.8s); POST /v1/auth/login → token + u-pro; GET /v1/portals, /v1/markets, /v1/jobs → correct camelCase rows matching frontend mappers
- 06:57 · RUN CLOSED — platform deployed at https://job-platform-api-mj5j.onrender.com
- 07:01 · frontend static site created via API: srv-daf62h67bikc73f7c4f0 → https://job-platform-web-h528.onrender.com (npm ci && npm run build, publishPath dist, VITE_API_BASE baked at build)
- 07:05 · SPA rewrite PUT /routes (/* → /index.html); CORS_ORIGINS updated to include frontend origin; backend redeploy dep-daf63e1t0dsc73cnqmq0 → live
- 07:12 · VERIFIED: / returns app shell (200, title served); deep route /some/deep/route → 200 (rewrite works); preflight OPTIONS from frontend origin → allow-origin echoed, authorization header allowed; credentialed GET /v1/markets with Bearer → 200
- 07:13 · FRONTEND+BACKEND DEPLOY RUN CLOSED — full stack live
