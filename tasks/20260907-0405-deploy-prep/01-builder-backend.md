# Task 01 — Builder: backend deploy-readiness for Render + Neon (v2)

## CRITICAL — state discipline
A previous builder died before writing ANY state. After EVERY numbered step
below, IMMEDIATELY append one status line to the `## Result` section at the
bottom of this file (replace PENDING with "in progress: <step> done"). Never
batch writes to the end.

## Repo
`/workspace/platform` (branch main, clean at 4476c37). Backend: `backend/`.
FastAPI app: `app/main.py` (already read for you — see Context). Package
manager: uv. Do NOT break the hermetic mock suite (85 tests).

## Context (already gathered — do not re-discover)
- `app/main.py` lines 28–36: `app = FastAPI(title="job-platform API", version="0.1.0", ...)`.
- Lines 40–46: `app.add_middleware(CORSMiddleware, allow_origins=["http://localhost:8080", "http://127.0.0.1:8080"], allow_credentials=True, allow_methods=["*"], allow_headers=["*"])`.
- Line 48: `API_PREFIX = "/v1"`; routers mounted in a for-loop after line 50.
- Migration script: `backend/scripts/migrate.py` (idempotent, reads DATABASE_URL + PG_SCHEMA env; default schema `jobplatform`).

## Work (in order)
1. **`/health`** in `app/main.py`, right after the `app = FastAPI(...)` block:
   ```python
   @app.get("/health")
   def health() -> dict[str, str]:
       return {"status": "ok", "version": app.version}
   ```
   Public, NO DB access. (Plain @app.get, not via a router — it must not sit
   under /v1.)
2. **CORS from env** — replace the hard-coded `allow_origins` list with:
   ```python
   import os
   _cors = [o for o in os.environ.get("CORS_ORIGINS", "").split(",") if o.strip()]
   app.add_middleware(
       CORSMiddleware,
       allow_origins=_cors or ["http://localhost:8080", "http://127.0.0.1:8080"],
       allow_credentials=True, allow_methods=["*"], allow_headers=["*"],
   )
   ```
   (`import os` goes at the top of the file with the other imports if not
   already imported.)
3. **`backend/render.yaml`**:
   ```yaml
   services:
     - type: web
       name: job-platform-api
       runtime: python
       plan: starter
       buildCommand: pip install uv && uv sync --frozen
       preDeployCommand: uv run python scripts/migrate.py
       startCommand: uv run uvicorn app.main:app --host 0.0.0.0 --port $PORT
       healthCheckPath: /health
       envVars:
         - key: DATABASE_URL
           sync: false
         - key: CORS_ORIGINS
           sync: false
         - key: PYTHON_VERSION
           value: "3.12"
   ```
4. **`backend/.env.example`** — placeholders ONLY, one-line comments:
   `DATABASE_URL` (Neon pooled connection string), `TEST_DATABASE_URL` (Neon,
   tests only), `PG_SCHEMA` (leave unset in prod — default jobplatform),
   `CORS_ORIGINS` (comma-separated frontend origins). Copy NOTHING from the
   real `.env`.
5. **`backend/DEPLOY.md`** (~40 lines): Render Blueprint walkthrough
   (New → Blueprint → repo pyth0nk3r/job-application-suite → set DATABASE_URL
   from Neon dashboard POOLED string → deploy → check /health); note
   migrate.py is idempotent per schema_migrations; interrupted-migration
   caveat (a deploy killed mid-migration can apply SQL without recording it —
   check `SELECT * FROM schema_migrations` before re-running).
6. **Verify** (append each result as you go):
   - `cd /workspace/platform/backend && uv run python -c "from app.main import app; print('import ok')"`
   - background `uv run uvicorn app.main:app --port 8971` (workspace_run_background), then a SHORT `curl -s http://127.0.0.1:8971/health`, expect `{"status":"ok",...}`, then workspace_background_kill.
   - hermetic mock suite: `cd /workspace/platform/backend && uv run pytest tests/ -q --ignore=tests/test_pg_drafts.py --ignore=tests/test_pg_fit.py --ignore=tests/test_pg_jobs.py --ignore=tests/test_pg_tracker.py --ignore=tests/test_db_pg.py` → 85 passed (~10s).
   - Do NOT run pg suites. Do NOT commit — orchestrator commits after critique; leave tree dirty.

## Output — final Result block format
```
## Result
Files: <path — one-line what> (one per line)
Verification: <command — outcome> (one per line)
Unverified: <list>
```

## Result (orchestrator-executed inline — see ESCALATION.md)
Files:
- backend/app/main.py — `import os`; CORS origins from `CORS_ORIGINS` env (empty → local-dev defaults, unchanged); DB-free `GET /health` → `{"status":"ok","version":"0.1.0"}`
- backend/render.yaml — Blueprint: uv build, migrate.py pre-deploy, uvicorn 0.0.0.0:$PORT, healthCheckPath /health, sync:false envVars
- backend/.env.example — placeholders only (DATABASE_URL, TEST_DATABASE_URL, PG_SCHEMA, CORS_ORIGINS)
- backend/DEPLOY.md — Blueprint walkthrough, pooled string, idempotent + interrupted-migration caveats
- frontend/src/lib/api/transport.ts — NEW: apiEnabled, apiFetch (15s abort, Bearer jp.auth, ApiError, 401→clearAuth), auth store helpers
- frontend/src/vite-env.d.ts — VITE_API_BASE typing
- frontend/src/lib/api/client.ts — login/logout/currentUser (mock fallback via mockLoginUser) + 3 flag-gated live reads (getJobs/getPortals/getMarkets with defensive mappers)

Verification: import ok; /health 200 live smoke; mock suite 85 passed; tsc clean; vitest 1/1; build ✓ 1m18s.
Unverified: live round-trip vs deployed backend (needs Render + user's Neon DATABASE_URL); prod CORS behavior; UI-level mock-identity smoke.
Commits: 61e2134 (backend plumbing), 809471f (frontend transport).