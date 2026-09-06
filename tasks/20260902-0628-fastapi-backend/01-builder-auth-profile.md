# 01 — Builder: Auth + Profile + Setup routers

Contract: `tasks/20260902-0628-fastapi-backend/00-CONTRACT.md` (read first).

## Deliverable
`backend/app/routers/auth.py`, `backend/app/routers/profile.py`, `backend/app/routers/setup.py` — plus tests in `backend/tests/test_auth_profile.py`.

## Endpoints

### auth.py — tag [Auth]
- `POST /auth/register` (no auth) → 201 AuthSession `{token, expiresAt, user}`. Body: RegisterRequest. 409 if email exists. Creates user (role "free") + token. Do NOT auto-seed jobs (seeding happens on first authed call via get_current_user).
- `POST /auth/login` (no auth) → 200 AuthSession. Body: LoginRequest. 401 invalid credentials.
- `POST /auth/logout` → 204. Reads the bearer token; `db.revoke_token`.
- `GET /me` → 200 User dict (the wire user).
- `PATCH /me` → 200 User. Body UpdateMeRequest (partial name/email). Email uniqueness: 409 if taken by another user.

### profile.py — tag [Profile]
- `GET /profile` → 200 profile dict. If none stored yet, derive a default from the user record (fullName=name, email, empty strings for optional text fields, workHistory=[], languages=[], cvLanguage="English") and store it before returning.
- `PUT /profile` → 200 saved profile. Body UserProfilePut; store as camelCase dict (field-by-field mapping, keep the derived defaults for absent optionals).
- `POST /profile/write-back-fact` → 200 `{ok: true}`. Body WriteBackFactRequest. Store the fact: append to `db.profiles[user_id]["_writeBackLog"]` (list of {fact, source, at}) — source defaults to "[chat]".
- `POST /profile/reset-sections` → 200 `{ok: true, reset: [...]}`. Body ResetSectionsRequest. For each named section in the user's setup_sections, set complete=False. Echo back the reset list. Unknown section names: skip silently (do not 404).

### setup.py — tag [Setup]
- `GET /setup/sections` → 200 list of SetupSectionState (from db.setup_sections).
- `POST /setup/sections/{id}/complete` → 200 updated list. 404 unknown id.
- `POST /setup/sections/{id}/reset` → 200 updated list (complete=False). 404 unknown id.

## Test file (backend/tests/test_auth_profile.py)
pytest style, use `fastapi.testclient.TestClient(app)` from app.main. Cover:
- register → 201 with token + user role "free"; duplicate email → 409
- login wrong password → 401; login correct → 200 with token
- /me without token → 401; with token → 200
- GET /profile returns camelCase shape (assert "fullName" in r.json())
- PUT /profile roundtrip (set location, read back)
- write-back-fact appends to _writeBackLog
- setup sections: GET has 9 entries; complete flips flag; unknown id → 404
- non-admin CAN reach these endpoints (they are not admin-gated)

Also create `backend/tests/conftest.py`:
```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture()
def client():
    return TestClient(app)

@pytest.fixture()
def user_headers(client):
    r = client.post("/v1/auth/login", json={"email": "user@demo.com", "password": "user123"})
    return {"Authorization": f"Bearer {r.json()['token']}"}

@pytest.fixture()
def admin_headers(client):
    r = client.post("/v1/auth/login", json={"email": "admin@demo.com", "password": "admin123"})
    return {"Authorization": f"Bearer {r.json()['token']}"}
```

## Done criteria
- `uv run pytest tests/test_auth_profile.py -q` green
- `uv run python -c "from app.main import app"` still imports (all routers exist or your import guards don't break others — main.py imports every router; if sibling routers don't exist yet, creating an EMPTY `backend/tests/conftest.py` as above is fine but do NOT create stub router files for other builders)

## Result
**Status**: DONE — all done criteria met.

**Produced**
- `backend/app/routers/auth.py` — tag [Auth]: POST /auth/register (201, 409 dup email, role "free", no eager seed), POST /auth/login (200/401), POST /auth/logout (204, revokes bearer token, 401 on unknown token), GET /me, PATCH /me (partial name/email, 409 email taken by another user, credential record re-keyed so login follows the new address). AuthSession wire shape {token, expiresAt (now+24h ISO), user}.
- `backend/app/routers/profile.py` — tag [Profile]: GET /profile (derives + stores default from user record on first call: fullName=name, email, "" optionals, workHistory=[], languages=[], cvLanguage="English", plus uploadedCvName/uploadedCvData="" per spec shape), PUT /profile (field-by-field camelCase via model_dump(by_alias=True); absent optionals keep derived defaults, "" text / "English" cvLanguage; preserves _writeBackLog), POST /profile/write-back-fact (appends {fact, source, at} to _writeBackLog; source defaults "[chat]"), POST /profile/reset-sections (flips complete=False for named setup sections, echoes actually-reset list, unknown names skipped silently).
- `backend/app/routers/setup.py` — tag [Setup]: GET /setup/sections, POST /setup/sections/{id}/complete (True), POST /setup/sections/{id}/reset (False); unknown id → 404.
- `backend/tests/conftest.py` — the specified client/user_headers/admin_headers fixtures, plus a guarded app resolution: uses the real `app.main` once all sibling routers exist; falls back to a partial app mounting only auth/profile/setup under /v1 while siblings are in flight (no stub router files created for other builders).
- `backend/tests/test_auth_profile.py` — 21 tests: register 201 shape + role "free", dup email 409, short password 422, login 401/200, /me 401 without token / 200 with, PATCH /me name update + email-collision 409 (on a dedicated user to avoid shared-state coupling), logout → token invalidated, profile default camelCase shape, PUT roundtrip (location read back; absent optionals keep defaults), write-back-fact append + default source, setup GET 9 sections, complete/reset flips, unknown section 404, reset-sections named-only + unknown skipped, free-role user reaches all endpoints (no admin gate).

**Verification** (exact contract commands, UV_CACHE_DIR=$PWD/.uv-cache UV_LINK_MODE=copy):
- `uv run python -c "from app.routers import auth, profile, setup"` → OK
- `uv run python -m py_compile app/routers/{auth,profile,setup}.py tests/{conftest,test_auth_profile}.py` → OK
- `uv run pytest tests/test_auth_profile.py -q` → **21 passed** (0 failed, 1 benign StarletteDeprecationWarning from TestClient itself)
- `uv run python -c "from app.main import app"` → still fails mid-run, ONLY on missing sibling routers (ImportError: cannot import name 'applications' from 'app.routers') — expected per contract; nothing I control, and my conftest guard keeps the suite green now and auto-switches to the real app when siblings land.

**Deviations**
1. conftest.py deviates from the literal snippet by wrapping the `app.main` import in try/except with a partial-app fallback (auth/profile/setup only). Required because main.py imports all 14 routers unconditionally and 11 sibling routers do not exist yet; without the guard the suite cannot even collect. The fixture API (client/user_headers/admin_headers) is identical to the spec.
2. PATCH /me re-keys the credential record in db.credentials inline (MockDB has no rename method and db.py is do-not-modify) so a changed email remains loggable. Documented as the mock-DB seam.
3. Profile default also includes uploadedCvName/uploadedCvData as "" — the task's default list omitted them but the spec's UserProfile has them; shapes must match the spec exactly.
4. One self-inflicted test-order bug found and fixed during verification (rename test mutated the shared demo user; moved to a dedicated account). Final suite green.

**Rating**: 9/10 — full spec coverage with exact response shapes, 21 green tests, zero edits to foundation files and no stubs for other builders. Point off only because the suite temporarily exercises a partial app rather than app.main until the sibling builders land.
