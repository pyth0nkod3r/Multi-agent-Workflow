# CONTRACT — FastAPI backend for job-platform (mock DB)

Read this + your task file only. Repo: /workspace/platform (monorepo).
Spec: /workspace/platform/docs/openapi.yaml (OpenAPI 3.0.3, 46 paths — the
contract). Implementation notes: /workspace/platform/BACKEND-NOTES.md.

## Foundation (already written — DO NOT MODIFY)
- `backend/app/schemas.py` — Pydantic v2 request models (camelCase wire via
  alias generator). Import what you need: `from app.schemas import ...`.
- `backend/app/db.py` — `MockDB` singleton `db` (in-memory, user-scoped,
  camelCase dict stores, seeded: demo users admin@demo.com/admin123 (role
  admin) + user@demo.com/user123 (role free), 8 jobs, 5 applications,
  portals/markets/templates globals, per-user scrape config + setup sections +
  automation + notion state). `db.ensure_user_seed(user_id)` is already called
  by `get_current_user` — you never call it.
- `backend/app/deps.py` — `get_db()`, `get_current_user` (Bearer), 
  `require_admin` (role admin/super_admin → 403 otherwise).
- `backend/app/main.py` — mounts ALL routers under `/v1`. Already imports
  every router file you or other builders write. Do not edit it.

## Router conventions (ALL builders)
- File: `backend/app/routers/<name>.py`. `router = APIRouter()` — no prefix,
  no tags kwarg needed (main.py adds prefix; add tags=[...] for docs).
  Actually: use `APIRouter(tags=["<TagName>"])` with the tag matching the
  OpenAPI spec's tag for your endpoints.
- Return plain dicts/lists in camelCase (they come from db stores already in
  wire shape — do NOT convert to snake_case).
- Every endpoint: `user: dict = Depends(get_current_user)` (or
  `require_admin` where the spec has x-admin-only: true — that marker exists
  on: /jobs/scrape, /jobs/portal-health, /tools/robots-check, /tools/portals,
  /tools/templates GET+POST, /tools/templates/{id}/activate, /portals/{portalId}
  PATCH. ADDITIONALLY admin-only (frontend AdminGuards the pages):
  /markets/{code}/activate, /markets/{code}/deactivate,
  /automation/config PATCH, /automation/test-connection, and ALL
  /integrations/* endpoints.
- Errors: HTTPException(404, "…") for missing resources; 403 via
  require_admin; 401 handled by deps.
- Mock behavior may be simplified BUT response SHAPES must match the spec
  exactly (field names, types). Requests validated with schemas.py models.
- Never import another builder's router.

## Verify before Result
`cd /workspace/platform/backend && export UV_CACHE_DIR=$PWD/.uv-cache UV_LINK_MODE=copy && uv run python -c "from app.routers import <your_module>" && uv run python -m py_compile app/routers/<your_module>.py`
uv only — never bare python. Then write `## Result` (status, produced,
deviations, rating N/10) into your task file, replacing the placeholder.
